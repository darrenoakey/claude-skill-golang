# Coding Rules & Standards

## 1. Naming & Style

-   **Exported**: `PascalCase` for types, functions, methods, constants, interfaces.
-   **Unexported**: `camelCase` for fields, local variables, helper functions.
-   **No abbreviations**: Write full words. `transaction` not `tx`. `context` not `ctx`. `database` not `db`. `definition` not `def`. `instance` not `inst`. `config` not `cfg`. The only exception is `id` which is a word in its own right.
-   **No consecutive capitals**: Acronyms are capitalized as words, not all-caps. `instanceId` not `instanceID`. `httpClient` not `HTTPClient`. `jsonData` not `JSONData`. `sqlQuery` not `SQLQuery`. This applies everywhere — fields, variables, function names, type names.
-   **Packages**: Lowercase, single word, no underscores, no plural (`text` not `texts`, `auth` not `authentication`).
-   **Interfaces**: Single-method interfaces named method + "er" (`Reader`, `Writer`, `Stringer`). Multi-method interfaces use a descriptive noun.
-   **Receivers**: Descriptive, not abbreviated. `store` for `Store`, `engine` for `Engine`, `server` for `Server` (not `s`, `e`, `srv`).
-   **Test Functions**: `Test[Function][Scenario]` in PascalCase. Example: `TestTruncateShortStringUnchanged`.
-   **Constructors**: `New[Type]` for constructor functions. Example: `NewService(logger Logger) *Service`.

## 2. Immutability & Value Semantics

-   **Unexported fields**: Struct fields are unexported by default. Expose via methods, not public fields.
-   **Constructor functions**: Use `New[Type]()` factory functions instead of allowing direct struct literal construction for types with invariants.
-   **Value receivers**: Use value receivers for methods that do not mutate state. Pointer receivers only when mutation is needed or the struct is large.
-   **Copy safety**: When returning slices or maps from a struct, return a copy, not the internal reference.
-   **Constants**: Use `const` for values known at compile time. Group related constants in a `const` block.

## 3. Typed Constants (Smart Enum Equivalent)

Go has no class-based enum. Use this pattern instead:

```go
// Status represents the processing state of an order.
type Status struct {
    value string
    code  int
}

// String returns the human-readable name.
func (s Status) String() string { return s.value }

// Code returns the numeric identifier.
func (s Status) Code() int { return s.code }

// IsTerminal reports whether the status represents a final state.
func (s Status) IsTerminal() bool {
    return s == StatusCompleted || s == StatusFailed
}

var (
    StatusPending   = Status{value: "Pending", code: 1}
    StatusCompleted = Status{value: "Completed", code: 2}
    StatusFailed    = Status{value: "Failed", code: 3}
)

// AllStatuses returns every defined status value.
func AllStatuses() []Status {
    return []Status{StatusPending, StatusCompleted, StatusFailed}
}
```

*See `status.go` example.*

## 4. Error Handling

-   **Return errors**: Every function that can fail returns `error` as the last value.
-   **Check immediately**: Never discard errors with `_`. Handle or propagate every error.
-   **Wrap with context**: `fmt.Errorf("fetching user %s: %w", id, err)`.
-   **Sentinel errors**: `var ErrNotFound = errors.New("not found")` for specific conditions callers check.
-   **Error types**: Implement the `error` interface when callers need structured details.
-   **No panic**: Never `panic()` for expected error conditions. Reserve for truly unrecoverable corruption.
-   **No log-and-return**: Either log the error OR return it. Never both (causes duplicate noise).

## 5. Documentation

-   **Mandatory**: Go doc comments on ALL exported types, functions, methods, and constants.
-   **Format**: Start with the name being documented. `// Truncate shortens a string to maxLen.`
-   **Content**: Explain **WHY**, not WHAT. The code shows what; the comment explains the reason.
-   **Package docs**: Add `doc.go` with a package comment when a package needs an overview.
-   **No stutter**: Do not repeat the package name. `// text.Truncate` not `// text.TextTruncate`.

## 6. Function Design

-   **Small**: Functions should be short enough to read without scrolling. ~25 lines max.
-   **Single responsibility**: Each function does one thing.
-   **Parameters**: Accept the narrowest interface that works. Return concrete types.
-   **Named returns**: Only use for very short functions where it aids clarity. Never use naked returns in functions longer than ~5 lines.
-   **Context**: Functions that do I/O or may block should accept `context.Context` as the first parameter. Exception: database operations — see Database Abstraction below.
-   **Options pattern**: For functions with many optional parameters, use the functional options pattern:
    ```go
    type Option func(*Config)

    func WithTimeout(d time.Duration) Option {
        return func(c *Config) { c.timeout = d }
    }

    func NewClient(opts ...Option) *Client { ... }
    ```

## 7. Concurrency

-   **No premature goroutines**: Do not add concurrency until profiling shows it is needed.
-   **Structured concurrency**: Every goroutine must have a clear owner, shutdown path, and error propagation.
-   **`errgroup`**: Use `golang.org/x/sync/errgroup` for coordinating goroutine groups with error handling.
-   **Channels for communication, mutexes for state**: Use the right tool.
-   **Context cancellation**: All long-running goroutines must respect `context.Context` cancellation.

## 8. Database Abstraction

Three concepts, one interface:

-   **Database**: Holds the connection string/pool. Has a `Transaction()` method that returns a Transaction. Implements the same query interface as Transaction.
-   **Transaction**: Holds an active transaction. Has `Commit()` and `Rollback()`. Defaults to rollback if not committed. Implements the same query interface as Database.
-   **The shared interface**: Both Database and Transaction implement it. All domain methods (`GetInstance`, `UpsertInstance`, `AddSetMembership`, etc.) live on this interface. Callers never know or care whether they're talking to a raw connection or a transaction.

```go
// Usage — callers just call methods on whatever they're given:
instance := transaction.GetInstance(id)
transaction.UpsertInstance(instance)
transaction.AddSetMembership(setPath, differentiator, instanceId)
transaction.Commit()

// Or without a transaction:
instance := database.GetInstance(id)
```

**Rules:**
-   No code outside the database layer knows about connections, transactions, or SQL details.
-   Domain methods do NOT accept `context.Context` — the database/transaction handles timeouts internally.
-   Only one or two places in the entire program create transactions. Everything else just receives "the thing that gets us to the database" and calls methods on it.
-   All operations within a logical unit of work (e.g., "cell resolves + propagate downstream") use the SAME transaction. Either everything commits or everything rolls back.

## 9. Prohibited Patterns (Zero Tolerance)

-   `interface{}` / `any` when a specific type or generic constraint works
-   `init()` functions (wire dependencies explicitly in `main`)
-   Global mutable state (package-level `var` that gets mutated at runtime)
-   `panic()` for expected error conditions
-   Naked returns in functions longer than ~5 lines
-   Discarding errors: `result, _ := something()`
-   Swallowing errors: `if err != nil { continue }` or `if err != nil { return }` without propagating. Either return the error or the system is lying about its state.
-   `time.Sleep` in tests for synchronization
-   Horizontal-layer package names: `utils/`, `helpers/`, `common/`, `models/`, `services/`
-   `TODO` comments
-   Mutable exported fields on structs
-   `log.Fatal` or `os.Exit` outside of `main()`
-   Abbreviations in identifiers: `tx`, `ctx`, `db`, `def`, `inst`, `cfg`, `stmt`, `buf`, `req`, `resp`, `conn`, `msg`, `val`, `idx`, `cnt`, `tmp`, `fn`, `cb`. Write the full word.
-   Consecutive capital letters: `ID` → `Id`, `HTTP` → `Http`, `URL` → `Url`, `JSON` → `Json`, `SQL` → `Sql`, `HTML` → `Html`, `API` → `Api`. Exception: `id` standalone (lowercase) is fine since it's a word.
-   Single-letter receivers: `s`, `e`, `c`, `r`, `w`. Use the full type name in lowercase.
