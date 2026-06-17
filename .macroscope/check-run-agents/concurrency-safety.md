---
title: Concurrency & Defensive Copying
model: claude-opus-4-6
effort: medium
input: full_diff
include:
  - "**/*.go"
exclude:
  - "**/*_test.go"
  - "**/*.pb.go"
  - "**/*_mock.go"
  - "api/**"
---

# Concurrency & defensive copying

Review the changed Go code for shared-memory safety. Focus on data that crosses a
lock boundary, a goroutine boundary, or a function boundary into a caller that may
read or mutate it concurrently. Browse the surrounding code to understand ownership
before flagging — only raise an issue when you can point at the shared reference.

### 🔴 Must fix — clone shared proto messages and maps; don't alias

Flag when a function returns, stores, or otherwise exposes a **proto message** or a
**map/slice** that is owned by mutable state, a cache, or anything held under a lock,
and hands out the *same underlying reference* instead of a clone.

- A proto pointer accessed outside the workflow/mutable-state lock must be cloned
  with `common.CloneProto(...)` (or the type's `CloneVT`/equivalent), not returned
  directly.
- A returned map/slice that the caller could read after the lock is released, or that
  the owner keeps mutating, must be copied.
- **A shallow copy is not enough when the values are still shared.** Allocating a new
  map but copying pointer values that still reference the originals (e.g. proto
  payloads) is *not* a defensive copy — the underlying objects remain aliased. Clone
  the values too.

### 🟡 Should fix — don't do IO while holding a lock

Flag network calls, disk/persistence calls, or other blocking IO performed while a
mutex is held. Prefer doing the IO outside the critical section (e.g. via a side
effect task), or clone the data you need and release the lock first.

### 🟢 Nit — prefer `sync.Mutex` over `sync.RWMutex`

Flag a newly introduced `sync.RWMutex` unless the code clearly has reads that vastly
outnumber writes, or readers that hold the lock for a long time. Default to
`sync.Mutex`.

## What not to flag

- Local values that never escape the function or the lock.
- Read-only/immutable values that are never mutated after construction.
- Returning a clone that is already produced via `CloneProto`, `maps.Clone`,
  `slices.Clone`, or an explicit element-wise copy.
- Generated code, mocks, and test files.

If the diff contains no shared-memory or locking concerns, **report nothing** — do
not manufacture findings on a clean PR.
