---
title: Go and API conventions
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*.go"
  - "**/*.proto"
exclude:
  - "**/*_test.go"
  - "**/*.pb.go"
  - "**/*_mock.go"
  - "**/mock_*.go"
---

# Go and API conventions

Review changes for the project's Go naming, error-handling, and proto/API design
conventions. These apply to **new or renamed** identifiers and **new or changed** public
surface — don't churn pre-existing names that the diff merely moves.

Report findings grouped by severity (🔴 Must fix / 🟡 Should fix / 🟢 Nit) with file and
line and the corrected form.

### 🟡 Go naming
- No `Get` prefix on getters: `func (a *Activity) Store()`, not `GetStore()`.
- No `Impl` suffix on implementation types.
- No underscore after `Test` in test names: `TestRetry`, not `Test_Retry`.
- Avoid stuttering: in package `activity`, name it `Status`, not `ActivityStatus`.
- Prefer the `value, ok := ...` boolean pattern over nil checks where idiomatic.

### 🟡 Error handling
- Prefer standard error types (`InvalidArgument`, `NotFound`, `FailedPrecondition`,
  etc.) over bespoke custom error types for handler/validation errors.
- Use `errors.AsType` instead of `errors.As` in this codebase.
- Don't `panic` in library code — return an error and let the caller decide. (Test
  helpers and genuine invariant violations via the project's `logger.Fatal`/`DPanic`
  conventions are the exception.)
- When wrapping an error, add real context: `fmt.Errorf("multi-operation part 2: %w",
  err)`. Flag wraps that add nothing (`fmt.Errorf("%w", err)`).
- Mark errors non-retryable when the task should not be retried on the queue.
- Validate inputs early in handlers, not deep inside business logic.

### 🟡 Proto / API design
- Every proto field needs a doc comment. Flag new fields in `.proto` files (and the
  generated structs they back) added without one.
- Proto field names must be snake_case: `request_id` not `requestId`, `schedule_time`
  not `scheduledTime`.
- Don't leak internal concepts into user-facing errors or messages (e.g. internal type
  names a caller can't act on).
- Prefer enums over loose int/string for well-known sets of values.

### 🟢 Minimize exported surface
Flag identifiers exported (capitalized) that are only used within their own package —
keep them unexported unless they need to cross the package boundary.

### Permission to do nothing
If the changed code follows these conventions, say so and report nothing. Do not flag
identifiers that are merely moved or reformatted without a name change, generated code,
or test files. When a rule is debatable for a given case, prefer 🟢 Nit or stay silent
over a confident false positive.
