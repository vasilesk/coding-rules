---
name: go-coding-conventions
description: "Go-specific coding conventions on top of `coding-conventions` — ctx propagation, pointer vs value receivers, mo.Option for optional values, named ID and enum types, an injected clock instead of time.Now, samber/lo over loops, error wrapping with gerund messages and sentinel errors, iter.Seq streaming in batch processing, testify suites and go.uber.org/mock, build tags for infrastructure tests, order of declarations in a file, params structs and functional options for partial updates. Use whenever writing, reviewing, or refactoring Go code (*.go, *_test.go). Not for package layout or dependencies between packages — use `go-hexagonal-architecture`."
---

Architecture rules (ports, where interfaces live, compile-time checks, dependency directions) are in the `go-hexagonal-architecture` skill; language-agnostic rules (objects and constructors, comment content, logging, early returns) are in `coding-conventions`. This skill covers what is specific to Go code.

- Codestyle:
  - Pass ctx through functions/methods to ensure proper cancellation propagation and deadline handling
  - Prefer pointer receivers for services, repos, mutable structs, etc.
  - Prefer value receivers for immutable structs
  - Prefer passing parameters by value rather than by pointer when possible
  - Use `mo.Option[T]` for optional values (fields, parameters, cursors) instead of pointers or zero-value sentinels; name variables holding an Option or Result with the `maybe` prefix (`maybeCursor`, `maybeDoc`)
  - Declare identifiers and enums as named types (`type DocumentID uuid.UUID`, `type Status string`) with `String()` and behaviour-describing predicates (`role.Superseded()`)
  - Declare fixed values of a type with an invariant (see `coding-conventions`) as follows:
    - anywhere: a package-level `var` initialized with `lo.Must(NewX(...))`; outside the type's package this is the only option
    - inside the type's package, also directly: a typed constant over a primitive (`const KeyVAT Key = "vat"`) or a package-level `var` with a struct literal, error values included. A conversion `Key(x)` is allowed only for a constant expression; anything else goes through the constructor
  - Do not call `time.Now()` in services or stores: inject a clock and read it once per operation, so one run sees one consistent "now" and tests control it
  - Prefer lo functions over explicit loops for pure transformations:
    - Use lo.Map for transforming slices
    - Use lo.Filter for filtering slices
    - Use lo.Reduce for aggregating values
    - Use other lo functions when appropriate (lo.GroupBy, lo.Uniq, etc.)
    - Use `lo/it` for the same operations over `iter.Seq`
    - Use a plain `for` loop when the body returns errors, `continue`s, or logs: forcing such code into lo callbacks hides control flow

- Comments:
  - Follow godoc conventions
  - Use // instead of /* */

- Error handling:
  - Always handle errors explicitly
  - Wrap errors with context, use gerund (ing-form)
```go
// ✅ Good
return fmt.Errorf("creating user: %w", err)
// ❌ Bad
return fmt.Errorf("failed to create user: %w", err)
```
  - Declare expected failures as sentinel errors (`var ErrNotFound = errors.New("...")`) with a comment saying when they are returned; callers classify with `errors.Is`
    - Exported sentinels live in the domain package and form the contract for callers
    - Unexported sentinels inside a service are fine for classifying outcomes internally (skip vs fail)
  - Introduce a custom error type only when callers need data from the error, not just its kind
  - Do not use panics. The only exception is `lo.Must` when initializing package-level fixed values (see Codestyle): it fails at program start, never while serving a request

- Batch processing:
  - Stream results through an iterator of value-or-error elements (`iter.Seq[mo.Result[T]]` or `iter.Seq2[T, error]`)
  - The iterator yields a failed page fetch as an error element; the consumer must stop on it, so a broken fetch never looks like an empty result
  - Consider logging and skipping a failure of a single item so that one bad record does not block the whole run, when the item can be safely retried by the next run. Decide per stage; when items depend on each other or partial results are harmful, fail the run instead

- Testing
  - Write unit tests in *_test.go files
  - Use table-driven tests where appropriate
  - Use testify: `suite` for tests sharing setup, `require` for preconditions whose failure makes the rest of the test meaningless, `assert` for independent checks
  - Pass the intent as the assertion message: `s.Equal(replacementID, to, "source items move to the replacement, not to the reversal")`
  - Generate mocks with `go.uber.org/mock` via a `//go:generate` directive next to the interface; put them into a sibling `mock` package
  - Put tests requiring real infrastructure behind a build tag (e.g. `//go:build db`)

- Database:
  - Use the SQL approach the bounded context already uses (sqlc, query builder, etc.); do not mix several in one store
  - Handle transactions explicitly

- File contents
  - The order of the file entities is the next: public interfaces, public types, public structs, private implementations of public interfaces (if the private implementation contains fields of private interface types, these types are described right above this private implementation)
  - When some struct is described, it is followed up by constructor, then public methods, then private methods
  - Avoid redundant package names in methods/types (e.g., use List() instead of ListObjects() in the object package, `taxdoc.Store` instead of `taxdoc.TaxDocumentStore`)
  - Use consistent parameter ordering (ctx first, ID second, options last)
```go
type Repo interface {
    Get(ctx context.Context, id UserID) (User, error)
    Create(ctx context.Context, user User) (UserID, error)
    List(ctx context.Context, ids []UserID) ([]User, error)
}
```

- External libraries:
  - github.com/samber/lo
  - github.com/samber/mo
  - github.com/stretchr/testify
  - go.uber.org/mock

- Function design:
  - Functions should use all passed parameters
  - Return fully populated structs (see `coding-conventions`):
```go
// ✅ Good
func ProcessUser(user User) (UserInfo, error) {
    return UserInfo{
        ID: user.ID,
        Name: user.Name,
        Status: calculateStatus(user),
    }, nil
}

// ❌ Bad
func ProcessUser(user User) (UserInfo, error) {
    info := UserInfo{
        ID: user.ID,
        Name: user.Name,
    }
    return info, nil // Status needs to be set externally
}
```
  - A constructor that guarantees an invariant returns an error; empty objects with errors are returned as `return MyStruct{}, err`. Without a constructor use a struct literal with named fields
```go
// ✅ Good
func NewPeriod(from, to time.Time) (Period, error) {
    if !from.Before(to) {
        return Period{}, ErrInvalidPeriod
    }

    return Period{From: from, To: to}, nil
}
```
  - Pass a parameters struct instead of a long argument list: when a function takes more than ~5 parameters, or several adjacent parameters of the same type that are easy to swap (`keysBatchSize, itemsBatchSize, workersCount int`; three `decimal.Decimal` amounts)
  - For partial updates use a params struct with `mo.Option` fields and functional options to set them; a None field is left unchanged
```go
type UpdateParams struct {
    Status       mo.Option[Status]
    CustomerInfo mo.Option[CustomerInfo]
}

store.Update(ctx, id, NewUpdateParams(WithStatus(StatusEnriched)))
```
