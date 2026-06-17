---
title: Concurrency safety
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*.go"
exclude:
  - "**/*_test.go"
  - "**/*.pb.go"
  - "**/*_mock.go"
  - "**/mock_*.go"
  - "api/**"
---

# Concurrency safety

Review changes for shared-mutable-state hazards: proto/map data that escapes a lock
while still aliasing memory another goroutine can mutate, IO performed under a lock,
and lock-type choices. Trace where a returned or stored value came from and who else
holds a reference to the same underlying memory.

Report findings grouped by severity (🔴 Must fix / 🟡 Should fix / 🟢 Nit) with file
and line, the data race or contention it creates, and the fix.

### 🔴 Shallow copy passed off as a defensive copy
This is the highest-value check. When code returns or exposes proto messages, maps, or
slices that are read or written by another goroutine (e.g. a `Describe`/`Get` handler
returning search attributes, memo, or headers; any value accessed **outside the
workflow lock**), flag it if the "copy" is shallow:

- A freshly allocated map/slice whose **elements still point at the same proto/payload
  objects** is NOT a defensive copy — mutating an element still races. Building a new
  `map[string]*commonpb.Payload` that reuses the same `*Payload` pointers is the exact
  bug reviewers have repeatedly caught here.
- Returning a struct/proto pointer directly (aliasing) when the caller may mutate it.

Require a real clone of the elements — `common.CloneProto(...)` (or the appropriate
deep-copy helper) on each proto value — not just a new container around the same
pointers. When you flag this, name which field is still aliased.

### 🟡 IO while holding a lock
Flag network calls, persistence/DB calls, RPCs, or other blocking IO performed while a
`sync.Mutex`/`RWMutex` (or the workflow lock) is held. It serializes unrelated work and
can deadlock. Move the IO outside the critical section (e.g. via a side-effect task), or
snapshot what's needed under the lock and do the IO after releasing it.

### 🟡 Mutating shared data after releasing the lock
Flag code that reads a pointer to shared data under a lock, releases the lock, and then
reads/returns that data assuming it is still consistent — if the data can be modified
concurrently, clone it **before** releasing the lock.

### 🟡 Lock-type choice
- Prefer `sync.Mutex` over `sync.RWMutex` unless reads vastly outnumber writes (roughly
  >1000×) or readers hold the lock for a meaningful duration. Flag a new `RWMutex`
  introduced without that justification.
- Prefer a `sync.Mutex` over hand-rolled `atomic` coordination unless there's a clear
  single-variable or performance reason; flag atomics used to guard multi-field state.

### Permission to do nothing
If the changed code introduces none of these hazards, say so and report nothing. Do not
flag established patterns that already clone correctly, generated code, or test files.
Do not speculate about races without pointing at the specific shared reference and the
goroutine that touches it.
