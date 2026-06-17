---
title: Go & API Conventions
model: claude-opus-4-6
effort: medium
input: full_diff
include:
  - "**/*.go"
  - "**/*.proto"
exclude:
  - "**/*.pb.go"
  - "**/*_mock.go"
---

# Go & API conventions

Review the changed code for a few specific, recurring project conventions around
error handling, dependencies, and proto design. Browse the codebase to confirm
existing patterns before flagging.

### 🟡 Should fix — error-handling conventions

- Flag use of `errors.As(...)` — this project uses the generic
  `errors.AsType[*T](err)` helper instead.
- Flag a newly introduced **custom error type** where a standard service error fits
  (`serviceerror.InvalidArgument`, `NotFound`, `FailedPrecondition`, etc.). Prefer the
  standard types for user-facing / RPC errors.
- Flag the pair `require.Error(t, err)` + `require.Contains(t, err.Error(), ...)` —
  collapse to a single `require.ErrorContains(t, err, ...)`.

### 🔴 Must fix — no new third-party dependencies without justification

Flag any new external module added to `go.mod` (a new `require` entry) or a new import
of a third-party package that the project doesn't already use, unless the PR clearly
justifies it. Small amounts of functionality should be written inline rather than
pulling in a dependency. Standard library and existing in-repo / already-used
dependencies are fine.

### 🟡 Should fix — document new proto fields

In `.proto` files, flag any **newly added field** (or new message/enum) that has no
leading doc comment. Every field should carry a short comment describing its purpose.

## What not to flag

- `errors.AsType` usage (that's the correct form), or comma-ok / sentinel-error
  checks via `errors.Is`.
- Re-use of error types and dependencies already established in the codebase.
- Generated proto Go (`*.pb.go`), mocks, and pre-existing fields you aren't changing.
- Linter-enforced rules already caught elsewhere (import aliases, `panic`,
  `pborman/uuid`, snake_case proto field names) — don't restate them.

If none of these conventions are touched by the diff, **report nothing**.
