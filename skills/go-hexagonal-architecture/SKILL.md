---
name: go-hexagonal-architecture
description: "Hexagonal architecture (ports and adapters) for Go projects sliced by bounded context — cmd/ and internal/ layout, domain, service, repo, client and handler packages, kernel/platform/pkg for shared code, allowed imports between layers, port interfaces with compile-time checks, DTOs at adapter boundaries, inprocess clients and workflow/ for cross-context calls, hand-written wiring in main.go, table ownership and migrations, go-arch-lint and depguard enforcement. Use when starting a Go service, adding a bounded context, package, repo, client, handler, or decorator, deciding where new Go code belongs, adding imports between internal packages, or editing .go-arch-lint.yml. Not for code style inside a file — use `go-coding-conventions`."
---

This project follows Hexagonal Architecture (Ports and Adapters), sliced by bounded context: each context owns its domain, application services, ports and adapters, so adding or removing a context touches one directory tree.

## Project Structure

```
.
├── cmd/
│   └── {app}/                       # entry point (http server, worker, cli); main.go holds the wiring
├── internal/
│   ├── kernel/                      # shared kernel: a directory of small packages, no code of its own
│   │   └── {concept}/               # one stable value type per package (e.g., money, country, apikey)
│   ├── platform/                    # in-house technical code shared by adapters of several contexts
│   │   └── {name}/                  # e.g., an external API transport, HTTP response helpers
│   ├── bootstrap/                   # optional: wiring helpers once main.go grows too large
│   ├── workflow/                    # optional: use cases that orchestrate several contexts
│   └── {context}/                   # bounded context (e.g., invoice, report)
│       ├── domain/                  # entities, value objects, domain errors
│       ├── service/                 # application services (use cases)
│       ├── repo/
│       │   └── {name}/              # secondary port: Repo interface and its types
│       │       ├── postgres/        # implementation
│       │       │   ├── sql/         # sqlc queries
│       │       │   └── query/       # sqlc-generated code
│       │       └── metrics/         # decorator implementing Repo
│       ├── client/
│       │   └── {name}/              # secondary port: Client interface and its types
│       │       ├── http/            # implementation over the network
│       │       ├── inprocess/       # implementation calling another context in the same binary
│       │       └── metrics/         # decorator implementing Client
│       └── handler/                 # primary adapters
│           ├── http/
│           └── consumer/            # e.g., queue consumers
├── pkg/                             # "external" packages: self-contained, could move to their own repositories
├── migrations/                      # single linear migration history for the database
├── config/                          # configuration files
└── script/                          # build, deploy and other scripts
```

The top level of `internal/` holds only bounded contexts and the fixed directories `kernel`, `platform`, `bootstrap` and `workflow`. Code that belongs to none of them is a sign of a missing context or of a misplaced package.

## Dependency Rules

Dependencies point inwards: adapters depend on the core, never the other way around.

| Package | May import |
|---|---|
| `kernel/{concept}` | stdlib, small utility libraries (uuid, lo, mo), other `kernel/*` packages |
| `{context}/domain` | stdlib, `kernel/*`, small utility libraries |
| `{context}/service` | stdlib, own `domain`, `kernel/*`, own port packages (`repo/{name}`, `client/{name}`, extension points), small utility libraries (including a clock), logging and tracing APIs |
| port packages (`repo/{name}`, `client/{name}`, extension points) | stdlib, own `domain`, `kernel/*`, small utility libraries |
| implementations (`postgres`, `http`, `metrics`, ...) | their port package, own `domain`, `kernel/*`, `platform/*`, `pkg/*`, infrastructure libraries |
| `{context}/client/{name}/inprocess` | the above plus another context's `service` and `domain` |
| `{context}/handler/*` | own `service`, own `domain`, `kernel/*`, `platform/*`, `pkg/*` |
| `platform/{name}` | stdlib, `kernel/*`, `pkg/*`, other `platform/*` packages, infrastructure libraries |
| `pkg/{name}` | stdlib, third-party libraries, other `pkg/*` packages; never anything from `internal/` |
| `workflow/*` | services and domains of the contexts it orchestrates, `kernel/*`, the libraries a service may use |
| `cmd/*`, `bootstrap` | anything |

- `service` never imports implementation subfolders (`postgres`, `http`, ...), only port packages.
- `domain`, `service` and port packages never import `platform/*` or `pkg/*`: the core does not know about transports.
- Logging, tracing and the clock are not transports: a service writes logs, opens spans and reads the time through the APIs of these libraries, injected by the wiring. Their setup (exporters, log sinks, observability middleware) is infrastructure and stays in `platform` and the wiring.
- `kernel/*` and `platform/*` import nothing from contexts.
- One context imports another only from `client/{name}/inprocess` or from `workflow`.
- Imports between project packages and the ban on transports in the core are enforced by linters in CI (see “Enforcing the Rules”). The rest of the rules — which third-party libraries a layer may use, data formats, services as the only entry to the core — are checked in review.

## Shared Code: kernel, platform, pkg

Code used by several contexts goes to one of three places, chosen by what it knows:

| Place | Contains | Knows about |
|---|---|---|
| `kernel/{concept}` | value types: identifiers, Money, Currency, Country, Address, an API key, an error contract shared by contexts | the language of the domain, nothing technical |
| `platform/{name}` | technical code: a transport to an external API, HTTP middleware and response helpers, observability setup | transports and this project's conventions (may use `kernel` types), never a specific context |
| `pkg/{name}` | self-contained libraries | nothing about this project |

- `kernel` is a directory of packages, one concept per package, named after the concept: `apikey.Key`, `money.Amount`, `country.Code`, not `kernel.APIKey`. There is no `kernel` package with code. A type that does not get a package with a clear name of its own does not belong to the kernel.
- Only small, stable types go to `kernel`, and their behavior means the same in every context: validating a country code, adding two amounts of one currency, formatting an address. Rules that belong to one context stay in its `domain`, even when they work on `kernel` types: how an invoice rounds its tax, which countries a product is sold in. A type without its own behavior is not a reason to create a new bounded context.
- `platform` is the place for code that every adapter of the same kind would otherwise duplicate: an HTTP client to an upstream API used by adapters of two contexts, the error-writing helper of HTTP handlers. Putting such code into `kernel` would let `domain` reach infrastructure; putting it next to the contexts blurs the top level.
- `pkg` holds "external" packages: code that could be moved to a separate repository as is and be imported back as a dependency. Such a package has a stable, documented API, its own tests, and no imports from `internal/`. Code that needs a `kernel` type or a project convention belongs to `platform`, not to `pkg`.
- When unsure between `platform` and `pkg`, start with `platform`: moving a package to `pkg` later only requires removing its `internal/` imports.

## Ports

- Secondary (driven) ports are the interfaces in `repo/{name}` and `client/{name}`. They are the full contract shared by the real implementation and its decorators (metrics, tracing, caching).
- Every implementation has a compile-time check: `var _ repo.Repo = (*repo)(nil)`.
- Consumers depend on the port interface directly. A narrower interface at the place of usage is declared only when the consumer needs a small subset of the methods.
- Primary (driving) adapters are the handlers. The primary port of a context is the set of exported methods of its services; there is no separate package of primary port interfaces. Each handler declares a small interface with the service methods it calls, at the place of usage. An explicit interface for a service is introduced only when the service gets several implementations (e.g., authorization or metrics decorators), and then it is an ordinary port.
- An extension point built as a chain of decorators (interceptors, middleware) is a port of its own, in a package next to `repo/` and `client/` (e.g., `{context}/interceptor/`), with implementations in its subfolders. Who assembles the chain depends on what the decorators do:
  - technical cross-cutting decorators (metrics, tracing, caching) carry no business meaning and are composed in the wiring;
  - decorators that carry business rules (e.g., request interceptors of a use case) are composed by the service: it receives them together with the port they wrap and builds the chain itself, because their order is part of the use case.

## Data Formats

- Domain types describe the business, not a representation: no struct tags, no `MarshalJSON`/`UnmarshalJSON`, no `Scan`/`Value`, no protobuf conversions. Encoding is part of a transport, and the core does not know about transports.
- Each adapter owns the format of its side: an HTTP handler owns the API the service exposes, a client implementation owns the API it calls, a `postgres` repo owns the table rows. The adapter declares types for its format next to the code that uses them and translates at the boundary. These are either DTO structs or, when the shapes match and the domain type is flat, a named type over the domain type with codec methods (`type wireModel domain.Model` with `MarshalJSON`, converted as `wireModel(m)`).
- A named type replaces the codec of its own level only: nested domain types keep their default encoding, so renaming a field of a nested type in the domain would silently change the format. A domain type that contains other domain structs, slices or maps of them is therefore always translated through DTO structs.
- Formats that coincide today are still declared separately in each adapter. The API a proxy serves and the API it calls are different contracts, and adding a second upstream with another format must not touch the domain or the other adapter.
- Data the core carries without interpreting it (fields a proxy forwards as is, a payload it relays) stays in the domain as opaque values (`json.RawMessage`, `[]byte`). The core neither parses nor produces them; only adapters do.
- Codec helpers shared by adapters go to `pkg` (a generic library) or `platform` (project conventions), and only adapters use them.

## Services

- A service holds a use case: it validates, orchestrates repos and clients, applies business rules, owns transactions.
- Handlers always reach the core through a service, even when a service method only forwards a call to a single port: the rule has no exceptions to check, and the logic that appears later has a place to go.

## Interaction Between Contexts

- Each context owns its representation of data. When context A needs data from context B, A describes it in its own terms (e.g., `Customer{ID, Name}` instead of B's whole `User`) and translates at the boundary.
- Synchronous calls go through a client port in the consuming context, implemented by an `inprocess` adapter that calls the other context's service and translates types. If the other context is later extracted into a separate service, only the adapter is replaced (`inprocess` → `http`).
- Asynchronous reactions go through events (e.g., transactional outbox), not direct calls.
- A use case that genuinely spans several contexts lives in `workflow/` and orchestrates their services, instead of one context driving another.
- A workflow reaches each service through a small interface with the methods it calls, declared at the place of usage, as a handler does. Its tests then mock these interfaces instead of assembling every context it orchestrates, and the interfaces show at a glance how much of each context the workflow relies on.
- Shared types go to `kernel`, shared technical code to `platform` (see above); neither is a way for one context to reach into another.

## Wiring

Wiring (the composition root) is the single place that knows which concrete implementations are used: it opens connections, creates implementations, wraps them with decorators, and injects them into services and handlers.

- Wiring lives in `cmd/{app}/main.go`; it moves to `internal/bootstrap` when it grows. Assembling servers and consumers from handlers (building the router, the `http.Server`) is wiring and lives there too, not in `platform`.
- Wiring is written by hand; no DI containers or code generators.
- Import aliases follow the `{context}{package}` pattern: `invoiceservice`, `invoicepostgres`, `invoicehttp`. Aliases are expected mostly in wiring, `inprocess` adapters and `workflow`.

```go
// cmd/httpserver/main.go
db := mustOpenDB(cfg.DB)
payments := paymentsapi.New(httpClient, cfg.Payments) // platform/paymentsapi, shared by two contexts

customerRepo := customermetrics.New(customerpostgres.New(db), registry)
customerSvc := customerservice.New(customerRepo, logger)

invoiceSvc := invoiceservice.New(
	invoicepostgres.New(db),
	invoiceinprocess.NewCustomerClient(customerSvc),
	invoicepayments.New(payments),
	logger,
)

srv := bootstrap.NewHTTPServer(cfg.HTTP, customerhttp.New(customerSvc), invoicehttp.New(invoiceSvc))
```

## Database

- Migrations live in the top-level `migrations/` as one linear history for the whole database.
- Every table is owned by exactly one context. Only that context's `postgres` adapters query it; other contexts get the data through a client port.
- sqlc queries live in `sql/` of the `postgres` adapter, generated code in its `query/` subfolder.

## Enforcing the Rules

- go-arch-lint checks imports between project packages (`.go-arch-lint.yml`). It maps every package to exactly one component and treats a package outside all components as an error, so a new directory cannot escape the rules.
- Its globs cannot express "the same context": `internal/*/domain` would put the domains of all contexts into one component. Each context therefore declares its own components, and a new context copies the block of an existing one:

```yaml
components:
  invoice-domain:   { in: internal/invoice/domain }
  invoice-ports:    { in: [internal/invoice/repo/*, internal/invoice/client/*] }
  invoice-service:  { in: internal/invoice/service }
  invoice-adapters: { in: [internal/invoice/repo/*/*, internal/invoice/client/*/*] }
  invoice-handlers: { in: internal/invoice/handler/* }

deps:
  invoice-domain:   { mayDependOn: [kernel] }
  invoice-ports:    { mayDependOn: [invoice-domain, kernel] }
  invoice-service:  { mayDependOn: [invoice-domain, invoice-ports, kernel] }
  invoice-adapters: { mayDependOn: [invoice-ports, invoice-domain, kernel, platform, pkg] }
  invoice-handlers: { mayDependOn: [invoice-service, invoice-domain, kernel, platform, pkg] }
```

- go-arch-lint does not check the standard library. Bans such as `net/http` in the core are a depguard rule in golangci-lint; it matches files by glob (`**/internal/*/domain/**`), so the rule needs no entry per context.
- Test files are excluded from both checks: tests may import whatever they need.
