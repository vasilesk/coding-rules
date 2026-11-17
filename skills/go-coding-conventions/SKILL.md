---
name: go-coding-conventions
description: "Go-specific coding conventions on top of `coding-conventions` — ctx propagation, pointer vs value receivers, mo.Option for optional values, named ID and enum types, an injected clock instead of time.Now, samber/lo over loops, error wrapping with gerund messages and sentinel errors, iter.Seq streaming in batch processing, testify suites and go.uber.org/mock, build tags for infrastructure tests, order of declarations in a file and splitting a package into files, named result parameters, params structs and functional options for partial updates. Use whenever writing, reviewing, or refactoring Go code (*.go, *_test.go). Not for package layout or dependencies between packages — use `go-hexagonal-architecture`."
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
    - Express each step of processing a collection (see `coding-conventions`) with its own lo function: a check of the input as a whole (`lo.FindDuplicatesBy`, `lo.ContainsBy`) goes before the transformation (`lo.KeyBy`), not into a loop that fills a map and fails on a repeated key
    - Use a plain `for` loop when the work on an item returns errors, `continue`s, or logs (parsing, a call): forcing such code into lo callbacks hides control flow. Move that work into a function processing one item and keep jumps out of the loop body as `coding-conventions` requires
```go
// ✅ Good
for _, row := range rows {
    if err := s.importRow(ctx, row); err != nil {
        return fmt.Errorf("importing row %d: %w", row.Line, err)
    }
}

// ❌ Bad: exits scattered among the work on an item
for _, row := range rows {
    amount, err := parseAmount(row.Amount)
    if err != nil {
        continue
    }
    entry := newEntry(row, amount)
    if _, ok := seen[entry.Key()]; ok {
        continue
    }
    seen[entry.Key()] = struct{}{}
    if err := s.store.Save(ctx, entry); err != nil {
        return fmt.Errorf("saving entry: %w", err)
    }
}
```

- Comments:
  - Follow godoc conventions
  - Use // instead of /* */
  - State what a package is for in the doc comment on its `package` clause (`// Package billing ...`), in one file of the package

- Error handling:
  - Always handle errors explicitly
  - Wrap errors with context, use gerund (ing-form)
```go
// ✅ Good
return fmt.Errorf("creating user: %w", err)
// ❌ Bad
return fmt.Errorf("failed to create user: %w", err)
```
  - Declare expected failures as sentinel errors (`var ErrNotFound = errors.New("...")`) with a comment saying what condition the error means, not which functions return it (a comment does not name its callers, see `coding-conventions`); callers classify with `errors.Is`. Which errors an operation returns belongs to the operation's own contract
```go
// ✅ Good
// ErrUnexpectedDocumentStatus means a document is not in the status an operation on it requires.
var ErrUnexpectedDocumentStatus = errors.New("unexpected document status")

// ❌ Bad
// ErrUnexpectedDocumentStatus is returned by the autotest pipeline stages when ...
```
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
  - Order declarations by their importance to a reader of the API, not by their kind: a file opens with what it exists for, and technical details come last
    - The file starts with its main exported entity: the type the file is named after or the package exists for (in a port package, the port interface). It is followed by its constructor, then exported methods, then unexported methods
    - Other exported types follow, each as a block: the type, its exported constants (enum values), its constructors and functions returning it (such as `Err` returning a field), then its methods
    - Unexported code goes last: helper functions, then unexported types with their methods (implementations of interfaces, `slog.LogValuer` and the like). An unexported interface used by a field of such a type is declared right above that type
    - An unexported constant or variable is declared right above its only user; one used in several places opens the unexported section. It never opens the file: a service value at the top hides the API behind it
  - Split a package into files by topic: the file named after the package holds its main type, and a cohesive group of supporting types with their helpers (`field.go` with `KeyValue`, `KV`, `Err` and the conversion to `slog.Attr`) may live in a file of its own. Do not split by kind (`types.go`, `consts.go`) or force one type per file: a file is read as one topic
  - Avoid redundant package names in methods/types (e.g., use List() instead of ListObjects() in the object package, `billing.Store` instead of `billing.BillingStore`)
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
  - Functions should use all passed parameters, and their parameter types should admit nothing the function cannot handle (see `coding-conventions`)
```go
// ✅ Good
type ChargeRequest struct {
    Amount money.Amount
    Card   CardToken
}

func (c *Client) Charge(ctx context.Context, req ChargeRequest) error

err := payments.Charge(ctx, ChargeRequest{Amount: order.Total(), Card: order.Card})

// ❌ Bad
// Charge expects the coupon to be applied and cleared by the pricing step.
func (c *Client) Charge(ctx context.Context, order Order) error
```
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
  - Do not name result parameters. A named result is a variable living through the whole body from its zero value: an assignment far from `return` or a bare `return` makes the reader trace the body to learn what the function returns, and a `:=` in an inner scope silently shadows it. Names are acceptable only to document several results of the same type (`(lat, lon float64, err error)`); even then they are documentation: do not assign to them, do not use bare `return`, return explicit values
  - A deferred function that changes the result (joining a `Close` error into `err`) is the only case where a named result is used as a variable. Keep such defers rare; the body still returns explicit values, which the defer then sees
```go
func export(path string, docs []Doc) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("creating file: %w", err)
    }
    defer func() { err = errors.Join(err, f.Close()) }()

    return writeDocs(f, docs)
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
