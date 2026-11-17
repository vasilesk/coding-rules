---
name: coding-conventions
description: "Language-agnostic coding conventions — DRY and single responsibility, early returns, continue/break/return inside loops, immutable objects with validating constructors, typed identifiers and enums, what comments should say, TEMPORARY markers, structured logging with snake_case keys and past-tense messages, keyset pagination for batch processing, separating infrastructure tests, database schema rules (table and index naming, UUID keys, NOT NULL, created_at/updated_at, no foreign keys), and environment-based configuration. Use whenever writing, reviewing, or refactoring code in any language, designing a table or migration, or adding comments, logging, or configuration. For Go code, load `go-coding-conventions` alongside."
---

- Common rules:
  - Follow DRY principles
  - Follow Single Responsibility principle
  - Depend on abstractions, not implementations; keep dependencies explicit and minimal
  - A signature accepts only what the function needs, in a type whose every value it can handle. When the caller's data carries more (a field meaningful only to another part of the code), declare a narrower type for the callee and convert at the call. Do not pass the wide type and expect someone before the call to have cleared the extra part: such a contract lives only in a comment, and a new caller or a changed order of calls breaks it silently at runtime, while a type mismatch fails to compile
  - Each service should have clear boundaries and responsibilities
  - Keep functions/methods short and focused.
  - Build the processing of a collection from small steps, each doing one thing over the whole input: validate, filter, transform, aggregate. A loop that checks, converts and accumulates at once mixes concerns and hides which step can fail; separate steps read one by one, and the pure ones have no error path. Keep a single pass when the steps cannot be separated without repeating work (parsing an item to check it, then again to use it) or when the input is a stream too large to traverse twice
  - Handle edge cases first with early returns, then write the main logic without nesting
  - Avoid `continue`, `break` and `return` inside a loop body. Most of them are a processing step in disguise: skipping an item is a filter, stopping at the first match is a search, and either can be written as a step of its own (see above). When a jump stays, it is the only statement of an `if` (a log line or a metric beside it is fine) whose condition is either simple or the result of a function that processes the item. That function holds the work on an item, where an exit is an ordinary early return, so the loop body stays a few lines. A jump buried among other statements makes the reader trace every path through the body to learn which items get processed and when the loop ends; a jump guarded by a plain `if` reads like an early return
- Objects:
  - Prefer immutable objects. Domain objects are immutable too: a change produces a new value or goes to storage as an explicit update, the object is not mutated in place
  - Prefer passing immutable parameters and receiving new values over mutating parameters
  - Rewrite data on its way only to change what it means (swap a key, pick another model), not to fit a consumer further on (clearing a field it cannot accept): that is the consumer's own type with a conversion at its boundary (see Common rules), and it does not depend on who runs before the consumer
  - Initialize an object in a single step after preparing all its fields
  - An object must be valid right after creation: validation is part of construction, not a separate IsValid-like method
  - Write a constructor when it adds something: validation, derived fields, generated values (IDs). A constructor that only copies its arguments into fields adds nothing over direct initialization with named fields
  - A function fully populates the data it returns. Filling an object step by step after creation is a rare exception (e.g. a builder local to one function), not a way to pass data between functions
  - Give identifiers and enumerations their own types instead of passing bare strings or UUIDs, so that values of different meaning cannot be mixed up
  - A type with an invariant beyond its base type (non-empty, a format, a range) has a constructor that validates it. Values known only at runtime (data from outside: gRPC/HTTP, the database, config, files; values assembled by code) are created only through that constructor: never by a literal, a zero value or a conversion from the base type (Key(s)), even where the language allows it. The type may stay a plain named type over a primitive: the rule is a convention, to be enforced by a linter. Absence of such a value is an optional, never the zero value.
  - Values fixed in source code (enum members, well-known values, error values) are an exception. Inside the type's package, which owns the invariant, they may be declared directly, bypassing the constructor; a test that runs each of them through the constructor is advisable. Anywhere, including outside that package, they may be built through the constructor once at program initialization, in a way that stops the program on failure, so an invalid value fails at startup, not while serving a request. Outside the type's package this is the only way. Either way the value is not built where it is used: a constructor call there adds an error path that cannot fire.
- Documentation:
  - All comments should be in English
  - Public API (exported functions, types, methods) must be documented. A method implementing a documented interface method is covered by the interface's comment (see below)
  - A comment explains what the code cannot say: the contract, invariants, why it is built this way in terms of the entity's own properties, non-obvious ordering, units, inclusive/exclusive bounds. Do not restate the name or narrate the body: a retelling adds nothing the code does not say and goes stale with the first change of the code
```
✅ Good: "the half-open window [From, To) of source item dates that one aggregation run merged into a document"
✅ Good: "linking source items runs before the number is reserved: it is the longest statement, and the reservation holds a lock other workers queue behind"
❌ Bad:  "aggregation period" (for AggregationPeriod)
❌ Bad:  "set document id"
❌ Bad:  "maps the error: an unknown ID first, then an entity the stage does not take, and anything else as a failure"
```
  - A comment says what the entity is and what it guarantees, never who uses it or which feature needs it: no callers, consumers, tests, services or tickets (except `TEMPORARY` markers). Check: the comment must stay true if a second, unrelated caller appears. Why the code exists in the project goes to the commit message or the MR description
```
✅ Good: "processes only the given IDs, so concurrent calls never touch each other's documents"
❌ Bad:  "so that parallel autotests do not process each other's documents"
❌ Bad:  "the outcome of an autotest pipeline stage"
```
  - Domain and service comments do not mention transport, clients or entry points (gRPC, HTTP, CLI): these layers do not know what sits above them, and their comments should not either
  - A comment outside a function body documents the declaration right below it, the module or package declaration included: what an operation does to the data goes on the operation, what a value holds goes on its type, what a package is for goes on the package. Documentation tools and IDE hovers show only such comments; a comment attached to no declaration is read only by someone scrolling the file
```
❌ Bad: "a failed document is not rolled back" on the result type instead of the stage that produced it
```
  - An interface method comment states the contract that holds for any implementation: inputs, outputs, errors, guarantees. An implementation does not repeat the contract; it documents only how the contract is achieved (transactions, batching, order of checks), and only when that is not obvious from the code
  - Mark temporary code with `TEMPORARY (<ticket>)`, the reason it exists and the condition under which it can be deleted
  - Include examples for non-obvious behaviors
  - All comments in function bodies (not in doc comments) should start with a small letter (except mentioning of some named entities)
- Logging:
  - Use structured logging
  - Include relevant context in log fields
  - Use snake_case field keys (`document_id`, `group_key`)
  - Write the message as an event that has happened, in the past tense: `proxy started`, `chat completion failed`, `validation error occured`. A log line records a fact, so failures and successes read alike; the error itself goes into a field, not into the message.
- Batch processing:
  - Process large result sets as a stream backed by cursor (keyset) pagination, instead of loading everything into memory or paginating by offset
- Testing:
  - Keep tests that need real infrastructure (database, network services) separate from unit tests, so the default test run stays hermetic
  - Add a message to assertions whose intent is not obvious from the compared values
- Database:
  - Use migrations for schema changes
  - Follow naming conventions:
     * Tables: plural, snake_case
     * Columns: snake_case
     * Indexes: idx_table_column
   - Use appropriate indexes
   - Use created_at and updated_at fields when creating tables, NOW() for DEFAULT for them
   - Use UUID type for identifiers, by default they should be uuid v4
   - Prefer NOT NULL where applicable
   - Prefer empty values over NULL values (especially for text columns)
   - Avoid adding/using foreign keys
   - Prefer TEXT over VARCHAR
- Configuration:
   - Use environment variables for configuration
   - Configuration structs hold primitive values as they come from the environment (strings, numbers, durations, URLs); values of domain types are built from them through their constructors where the application is wired. An optional setting is an optional, not an empty value.
