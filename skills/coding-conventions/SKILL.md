---
name: coding-conventions
description: "Language-agnostic coding conventions — DRY and single responsibility, early returns, immutable objects with validating constructors, typed identifiers and enums, what comments should say, TEMPORARY markers, structured logging with snake_case keys and past-tense messages, keyset pagination for batch processing, separating infrastructure tests, database schema rules (table and index naming, UUID keys, NOT NULL, created_at/updated_at, no foreign keys), and environment-based configuration. Use whenever writing, reviewing, or refactoring code in any language, designing a table or migration, or adding comments, logging, or configuration. For Go code, load `go-coding-conventions` alongside."
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
  - Public API (exported functions, types, methods) must be documented
  - A comment explains what the code cannot say: why, the contract, invariants, non-obvious ordering, units, inclusive/exclusive bounds. Do not restate the name
```
✅ Good: "the half-open window [From, To) of source item dates that one aggregation run merged into a document"
✅ Good: "linking source items runs before the number is reserved: it is the longest statement, and the reservation holds a lock other workers queue behind"
❌ Bad:  "aggregation period" (for AggregationPeriod)
❌ Bad:  "set document id"
```
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
