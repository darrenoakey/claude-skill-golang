# Go Gotchas & Known Pitfalls

## Gio GUI Framework
See `GIO_GOTCHAS.md` for all Gio-specific pitfalls (theme allocation, frame handler deadlocks, key/pointer filters, clipboard, display link, etc.).

## CGO & Cross-compilation
- `go:embed` embeds files into the binary at compile time. Changes to embedded files (HTML/CSS/JS in dashboards) require rebuilding the binary and restarting the process to take effect. The embedded files are NOT read from disk at runtime.
- **Platform stubs, cross-compilation, Linux CI headers**: See `GIO_PLATFORM.md` for detailed examples and CI configs.

## Process Management
- Process group killing: when processes are started with `setsid()`, killing the parent with `os.Kill` leaves orphaned children. Use `syscall.Kill(-pgid, syscall.SIGTERM)` or `unix.Kill(-pgid, unix.SIGTERM)` to kill the entire process group.
- Distinguishing intentional process exit from crash/reboot: use a `startupComplete` flag set after initialization. If a resource dies while `startupComplete == true`, it was intentional → clean up persisted state. If the process is killed before setting the flag, state persists for recovery on restart.
- When testing "process killed abruptly" (reboot scenarios): nil out all exit callbacks before killing the target. Real OS kills fire no Go callbacks; if you kill without nil-ing, callbacks run, config gets cleaned, and recovery never triggers.

## MCP / SDK
- MCP Go SDK v0.2.0 (`github.com/modelcontextprotocol/go-sdk`): the `AddTool` generic handler signature is `func(ctx context.Context, ss *mcp.ServerSession, params *mcp.CallToolParamsFor[Args]) (*mcp.CallToolResultFor[Out], error)` — NOT `func(ctx, *mcp.CallToolRequest, args)` as some web docs show. Access args via `params.Arguments`. `CallToolResult` is an alias for `CallToolResultFor[any]`.

## Testing
- macOS `/var`, `/tmp`, `/etc` are symlinks to `/private/var`, `/private/tmp`, `/private/etc`. Tests comparing filesystem paths (e.g. from tmux `pane_current_path`, `os.Getwd()`, or subprocesses) must use `filepath.EvalSymlinks` on both sides before comparing, or use real paths like `/usr/local`. The resolved path is the one tmux/OS returns.
- Use `t.Cleanup(func() { /* teardown */ })` not deferred-after-Fatal for tmux/subprocess cleanup. `t.Fatalf` calls `runtime.Goexit()` skipping deferred code — `t.Cleanup` runs regardless.
- Test servers with hardcoded ports fail when multiple test runs execute concurrently. Always use `net.Listen("tcp", "127.0.0.1:0")` to get an OS-assigned free port.
- **Goroutine timer test poisoning**: `time.AfterFunc`/goroutines spawned during a test that reference shared state (mutexes, App pointers, session objects) can fire AFTER the test completes and poison subsequent `-count=N` runs. Orphan goroutines hold references to defunct state and contend on locks. Prefer coalescing logic in the render/poll path over background timer goroutines that outlive the test scope.
- `os.Setenv`/`os.Unsetenv` in tests: defer cleanup must ALWAYS unset when the original was empty, not just skip the restore. Pattern: `if orig != "" { os.Setenv(key, orig) } else { os.Unsetenv(key) }`. Skipping the `else` branch leaks test values into subsequent tests in the same package (Go runs tests sequentially within a package).
- OpenAI SDK `apierror.Error.Error()` panics when `.Request`/`.Response` are nil (e.g., in unit tests constructing typed errors). Check with `errors.As` before calling `.Error()` in error classification code.

## Lambda / Deployment
- Compile with `-ldflags="-s -w"` to strip debug symbols. Lambda's 6MB limit applies to the **compressed** zip, not raw binary.

## Runtime
- Long-running daemons: set `debug.SetMemoryLimit(N)` at startup. Without it, Go GC only triggers when heap doubles — transient bursts can balloon to OOM. Set to ~2.5x expected steady-state RSS.
- **Adaptive schedulers must distinguish inspected work from actual progress.** A dependency-blocked item that returns `true`/"worked" after deciding to skip pins the scheduler at its minimum delay forever: the same blocked rows are reopened, reclassified, and committed in a hot loop even though no result changes. Return no-progress for unmet dependencies/passive skips, let the idle backoff grow, enumerate only state/time-eligible levels instead of `0..maxHistoricalLevel`, and regression-test a permanently blocked item plus the scheduler's work result.
- `macOS filepath.EvalSymlinks` on non-existent paths: `/var/folders` is a symlink to `/private/var/folders`. Walk the ancestor chain to find the first existing directory, resolve it, then reattach remaining components.
- gopsutil on macOS: `p.Name()` returns "Python" (capital P). Use `strings.ToLower` for interpreter matching. Use `p.Cwd()` for meaningful process identification.
- Unrecovered panic in ANY goroutine kills the entire process. Always add `defer func() { if r := recover(); r != nil { log(r) } }()` in fire-and-forget goroutines.

## Templates / Web
- `html/template` with `embed.FS`: use `fs.Sub(staticFSRaw, "static")` before `http.FileServer`. Multiple template files defining `{{define "content"}}` conflict — parse layout once, then `Clone()` per page.
- **Per-request template building** (more robust than `Clone()`): when multiple page templates all define `{{define "content"}}`, the last one parsed wins globally. The fix is `buildTemplate(pagePath)` which creates a fresh `template.Template` per render: parse base.html + partials + target page template into a new set each time. Pre-read all template text at init; per-render cost is in-memory string parsing only (microseconds). `Clone()` fails for the same reason — the cloned set still has all page defines registered.
- SSE endpoints (`text/event-stream`) never close — Playwright's `networkidle` hangs forever. Delay `new EventSource()` in JS so the page's `load` event fires first.
- **`go:embed` path restriction**: `//go:embed` cannot traverse `..` — all embedded files must be in a subtree under the package containing the directive. Move templates and static assets into a dedicated `internal/assets/` package with its own `assets.go` containing `//go:embed templates static`.

## Playwright-Go (playwright-community/playwright-go)
- **`HasText` field type**: `LocatorFilterOptions.HasText` is `any` — it type-asserts to `string` internally. Pass a bare `string` value, NOT `playwright.String(text)` which returns `*string` and causes `interface{} is *string, not string` panic.
- **`SelectOption` returns `([]string, error)`**: not just `error`. Always capture with `_, err := locator.SelectOption(...)`.
- **`WaitForNavigation` does not exist** in v0.5700.1: use `page.WaitForLoadState()` instead.
- **`BrowserNewContextOptions.ViewportSize`** does not exist: use `Viewport *playwright.Size` field. `playwright.SetViewportSize` also does not exist — construct `playwright.Size{Width: W, Height: H}` directly.
- **Filter select + JS `form.submit()` race**: selecting a filter value via `SelectOption`, then submitting via `page.Evaluate("form.submit()")` triggers navigation. If you then select another filter on the resulting page, you get "Execution context was destroyed". Fix: read current filter values from the DOM, build the URL with all params, and call `page.Goto(url)` directly instead of submitting a form.
- **Comment POST redirect + immediate assertion**: after `locator.Click()` on a submit button that triggers a POST+redirect, `WaitForLoadState()` is not sufficient to guarantee the new comment appears. After the redirect completes, also wait for the new comment element: `commentLocator.WaitFor(LocatorWaitForOptions{State: Visible})`.
- **Strict mode violation with shared `data-test` selectors**: if both list panel and detail panel have `[data-test="empty-state"]`, Playwright strict mode fails with "2 elements matched". Scope the selector: `.issue-list-scroll [data-test="empty-state"]`.

## SQL / database/sql
- **An index cannot make an unfiltered full-table `JOIN ... GROUP BY` low-order.** For exact hot-path counts, maintain a compact aggregate ledger transactionally as base rows change (database triggers are appropriate when many write paths exist), then query only the ledger. Preserve every filter semantic as an explicit ledger dimension (for example, cell tombstone versus membership tombstone), use a durable version marker so the base-table backfill runs exactly once rather than at every startup, and regression-test both ledger/base equality and `EXPLAIN` plans that do not reference the large base tables.
- **An idempotent startup schema is not permission to rerun historical data migrations.** `CREATE TABLE/INDEX IF NOT EXISTS` is safe to repeat, but an old `UPDATE`, `DELETE`, cleanup, or backfill still scans current-sized tables on every restart—even when there is no legacy data left. Guard each data migration with a durable version row and put the existence check outside every expensive statement. Add indexes needed by the migration or hot delete path before the table grows; a missing foreign-key lookup index turned a long-finished cleanup into a 68-million-row restart scan.
- `json.RawMessage` (`[]byte`) cannot scan NULL from PostgreSQL — use `sql.NullString` instead when a LEFT JOIN may produce NULL columns. Convert with `json.RawMessage(nullStr.String)` after checking `.Valid`.
- `github.com/jackc/pgx/v5/stdlib` through `database/sql` may scan PostgreSQL array columns (e.g. `BIGINT[]`) as Postgres array text such as `{1,2}`, not into `pgtype.Array[T]`. For portable reads, select `array_to_json(col)::text` and `json.Unmarshal` into `[]T`; for writes, set `pgtype.Array` dimensions explicitly or verify empty arrays do not encode as SQL NULL.
- **SQLite/PostgreSQL dual-driver pattern**: Write all SQL with `?` placeholders, convert to `$1, $2, ...` for PostgreSQL via a helper. Both `modernc.org/sqlite` (driver name `"sqlite"`, pure Go, no CGO) and `github.com/lib/pq` (driver name `"postgres"`) support `ON CONFLICT ... DO UPDATE SET` for upserts.
- For SQLite via `database/sql`, call `db.SetMaxOpenConns(1)` to avoid concurrent write issues.
- With `database/sql`, do not issue child queries while a parent `*sql.Rows` is still open if the pool may be pinned to one connection (`SetMaxOpenConns(1)`, transaction-like tests, SQLite). Scan parent rows into memory, check `rows.Err()`, close `rows`, then run follow-up queries; otherwise the child query can hang forever waiting for the only connection held by the open cursor.
- **Prepared statements for remote PostgreSQL**: Query planning adds ~25-30ms overhead per call. For hot-path queries called 1000+ times, use `db.Prepare()` at init and cache the `*sql.Stmt` on the Store struct. Cuts per-call latency in half for remote databases.
- **Batch DB writes into transactions**: 100 operations in one `BeginTx/Commit` is ~100x faster than 100 auto-commit calls (1 network round-trip + 1 fsync vs 100 of each). Use `WriteItemResult`-style methods that bundle value insert + state update + execution record + event in a single tx.
- **CTE planning overhead**: Complex CTEs with UPDATE...RETURNING pay ~30ms PostgreSQL planning time per call. Prefer simple UPDATE...FROM + separate INSERT over a CTE when called in tight loops.
- **sqlite-vec distance metric is L2, not cosine**: A `vec0` virtual table created with bare `embedding float[N]` (no column-level `distance_metric=`) reports squared L2 distance from `WHERE embedding MATCH ?` queries. For L2-normalized embeddings (ArcFace/CLIP/etc.), convert back to cosine with `cos = 1 - d²/2`. Silently using `distance` as if it were cosine will mis-rank clusters and break similarity thresholds.
- **sqlite-vec + mattn/go-sqlite3 on macOS**: `sqlite_vec.Auto()` compiles fine but emits `sqlite3_auto_extension is deprecated` warnings from Apple's SDK headers. They're non-fatal — the extension loads correctly and `vec_faces` tables work. Don't chase the warning.
- **sqlite-vec Go binding choice**: Use `github.com/asg017/sqlite-vec-go-bindings/cgo` paired with `github.com/mattn/go-sqlite3` (CGO). The pure-Go `modernc.org/sqlite` driver can't load arbitrary C extensions, so sqlite-vec is not compatible with it.

## chromedp
- Chrome reattach: read `DevToolsActivePort` from user data dir, verify via `/json/version`, connect with `chromedp.NewRemoteAllocator`. Falls back to launching fresh.
- `chromedp.Run()` requires context from `chromedp.NewContext()`. Passing `context.Background()` gives "invalid context".
- CDP protocol calls (`.Do(ctx)`) require `cdp.WithExecutor()` — wrap in `chromedp.Run(ctx, ActionFunc(...))`.
- `chromedp.Evaluate` with async IIFE returns the Promise object — use `runtime.Evaluate` with `WithAwaitPromise(true)` + `WithReturnByValue(true)`.
- `input.DispatchKeyEvent` with modifier bitmasks is silently ignored by many web apps. Use `chromedp.KeyEvent("D")` (uppercase for shift).
- `chromedp.FullScreenshot` with quality param produces JPEG — use `page.CaptureScreenshot` with explicit PNG format.

## JWT / Waft Authentication
- Hand-rolled HS256 JWT using only stdlib (`crypto/hmac`, `crypto/sha256`, `encoding/base64`, `encoding/json`). No external JWT library needed. `base64url` encoding must strip `=` padding to match Python's `urlsafe_b64encode().rstrip(b"=")`.
- Test-mode auth bypass pattern: set `Config{TestMode: true}` and have middleware check for `X-Waft-Test-Auth: {"user_id":"...","email":"..."}` header. Eliminates real OAuth in all tests without mocks.
- `auth.UserStore` interface with `GetPage(slug) string` (body only) vs `store.Store.GetPage() *model.Page`: bridge with an adapter that extracts `.Body`. Use a `pageOnlyAdapter` (other methods panic) when only role resolution is needed.

## Circular Import Resolution
- When a `capability` subpackage imports the root package for types, the root package cannot import `capability` back. Solve with injectable function fields on structs (e.g., `Agent.ImageFn func(...)`) — the CLI or main wires up both packages. Cleaner than interfaces for a small number of capabilities.

## Ollama Image Generation
- Ollama supports image gen models (z-image-turbo, flux2-klein) via standard `/api/generate` endpoint. Response has `image` field with base64 PNG data (not in `response` field). Model names are prefixed with `x/` (e.g., `x/z-image-turbo`). macOS only as of early 2026.

## Filesystem
- `os.MkdirAll` fails on paths with symlink components (e.g. `~/big2` → `/Volumes/big2`). Use `filepath.EvalSymlinks` on the parent dir first.

## Gmail API
- `list-messages` with `labelIds=INBOX` + `q=after:TIMESTAMP` triggers `failedPrecondition`. Use `search` with query `"in:inbox after:N"`.
- Query parameters with spaces MUST be URL-encoded with `url.QueryEscape()`. Unencoded spaces cause HTTP 400 `failedPrecondition`.

## PostgreSQL JSONB ↔ Go Hash Matching
- PostgreSQL `jsonb::text` and Go `json.Marshal(map[string]interface{})` both produce compact JSON with space after colon (e.g., `{"a": "b"}`).
- Go-compatible SHA-256 in PostgreSQL: `left(encode(digest(convert_to(data::text, 'UTF8'), 'sha256'), 'hex'), 16)` matches `fmt.Sprintf("%x", sha256.Sum256(jsonBytes)[:8])`.
- Requires pgcrypto extension (`digest` function).
- Useful for bulk migrations where value_versions hashes need recomputing without Go.

## JSON / Config
- Go's `int` zero value is 0, same as JSON `0`. If 0 is a valid API value (e.g., `max_instances: 0` meaning "disabled"), use `*int` so `nil` = "not set" and `0` = explicitly set. Without the pointer, `json.Unmarshal` can't distinguish between "field omitted" and "field set to 0".
- When a subprocess-spawning Go server uses VRAM bookkeeping (e.g., `usedGB`), every code path that frees resources MUST decrement the counter. If model loads succeed but inference fails and the instance restarts, the VRAM from the failed load may not be released — leading to phantom VRAM usage that accumulates on each crash/recovery cycle. Add safety caps (`if condemned > used { condemned = used }`) as a backstop.

## Temporal
- Go SDK is `go.temporal.io/sdk` (the de-facto standard; SDK ~v1.44, API `go.temporal.io/api`). Register workflows + activities on a `worker.New(c, taskQueue, ...)`; test workflows with `testsuite.WorkflowTestSuite` (real activities, no mocks). `client.GetWorkflowHistory(...HISTORY_EVENT_FILTER_TYPE_ALL_EVENT)` + the `ActivityTaskScheduled`/`ActivityTaskCompleted` events (linked by `ScheduledEventId`) reconstruct per-step inputs/outputs. `converter.GetDefaultDataConverter().ToStrings(payloads)` returns JSON-encoded strings — a string payload comes back quoted (`"x"`); `json.Unmarshal` into a string to unquote for display.
- Running Temporal against an EXTERNAL Postgres: `temporal server start-dev` is SQLite-only and can't point at Postgres. Use the `temporalio/auto-setup` Docker image instead (env `DB=postgres12`, `DB_PORT`, `POSTGRES_USER`, `POSTGRES_PWD` (set even for trust auth — it's sent then ignored), `POSTGRES_SEEDS=<host>`); it creates the `temporal` + `temporal_visibility` DBs and installs schema (incl. advanced visibility: `search_attributes JSONB` + `btree_gin`) on first boot.
- The `temporalio/auto-setup` image tag LAGS the CLI's bundled server version — `temporalio/auto-setup:<server-version>` often 404s. Check Docker Hub tags (`curl -s "https://hub.docker.com/v2/repositories/temporalio/auto-setup/tags?ordering=last_updated"`) and pin the latest real tag; the Go SDK is version-tolerant across nearby server versions. Isolate test runs from app runs with separate Temporal namespaces on the same server (e.g. `beezle` vs `beezle-bdd`).

## Postgres / Temporal activities
- `CREATE TABLE IF NOT EXISTS` is NOT safe under true concurrency: parallel connections (e.g. separate `go test` package processes sharing one DB) race and fail with `duplicate key value violates unique constraint "pg_type_typname_nsp_index"` (SQLSTATE 23505) on the implicit row type. Fix: wrap the DDL in a tx that first takes a `SELECT pg_advisory_xact_lock(<const>)` so schema creation serializes DB-wide across processes.
- Temporal struct-based activities: register an instance (`w.RegisterActivity(&Activities{deps})`) so methods can hold dependencies (DB, clients). In the WORKFLOW (which has no instance) reference them via a nil typed pointer — `var a *Activities; workflow.ExecuteActivity(ctx, a.Method, arg)` — the SDK resolves the registered name from the method expression without dereferencing. Lets you give one activity a DB while keeping others pure.
- Integration tests that hit a shared Postgres: isolate by DATABASE (point `*_DSN` at a `*_test` DB via t.Setenv), not by mocking — keeps the app DB clean while still exercising real SQL. Same pattern as Temporal namespace isolation (app vs `*-bdd`/test namespace on one server).
- Editing a Go package inside a `git worktree` while the editor LSP is rooted at the MAIN checkout: gopls floods `BrokenImport` ("could not import X in GOROOT") + `undefined` errors + "This file is within module '../<worktree>/src', which is not included in your workspace" for every file in the worktree. These are FALSE POSITIVES — gopls only indexes the primary checkout's module, not the sibling worktree, so it can't resolve any import or same-package symbol there. The authoritative check is the CLI: `cd <worktree>/src && go build ./... && go vet ./...`. If those pass, ignore the LSP entirely. (To actually silence it you'd add a `go.work` covering both dirs, but that's rarely worth it for short-lived agent worktrees.)
- Multi-module repo (e.g. a separate `tests/bdd` / e2e Go module with its own go.mod + `replace ../src`): `go vet ./...` / `go build ./...` run from the MAIN module do NOT cover the sibling module, so changing an exported signature in the main module (e.g. adding a param to `runner.Build`) compiles + vets clean there while the e2e module still calls the old form. A CI gate that vets every module then fails late on the sibling. When you change any exported signature, grep ALL modules for callers and `cd <other-module> && go vet ./...` before declaring clean. (Subagents editing one module are especially prone to missing this.)
- Temporal long-lived ENTITY workflow + code change = replay non-determinism. Adding/removing/reordering an activity (or any command) in a workflow that is ALREADY RUNNING breaks replay of its recorded history under the new code: the worker panics every task with `[TMPRL1100] lookup failed for scheduledEventID to activityID: scheduleEventID: N` and retries FOREVER (the entity is wedged; signals pile up unprocessed → callers "hang"). Especially bites singleton/entity workflows (per-conversation inbox, merge queue) that live across deploys. Fixes: (1) gate new commands behind `workflow.GetVersion(ctx, "change-id", workflow.DefaultVersion, 1)` so old histories replay the old path; (2) frequent `ContinueAsNew` bounds history but does NOT make an in-flight change safe; (3) emergency recovery — `temporal workflow terminate --workflow-id <id>` the wedged instance so it recreates fresh under current code (durable side-effects must live in your DB, not workflow state, so nothing real is lost). Treat entity-workflow logic changes like a schema migration.
- Temporal long-lived ENTITY workflows (a singleton that runs ~forever via signals + ContinueAsNew): adding/removing an activity in its loop AFTER instances are live makes replay diverge → `[TMPRL1100] lookup failed for scheduledEventID to activityID` panic on EVERY task → the workflow wedges + retries forever (and any signals it holds — e.g. queued chat turns — never process). Guard every post-hoc change with `workflow.GetVersion(ctx,"name",DefaultVersion,N)` so old histories replay the old path. Recovery for an already-wedged instance: `temporal workflow terminate` it (state in your own DB survives); it recreates fresh on the next signal under the current code.
- Temporal activity RetryPolicy with NO `MaximumAttempts` retries FOREVER (InitialInterval→MaximumInterval backoff). For a BEST-EFFORT enrichment activity (logged + swallowed by the workflow) that's a footgun: a persistently-failing call becomes a storm. Best-effort activities want `RetryPolicy{MaximumAttempts:1}` + a bounded StartToCloseTimeout; reserve retrying policies for activities whose durable redrive you actually want.
- Wrapping expensive/external CLIs (paid LLM agents like claude/codex/gemini, slow GPU tools) as Temporal activities or library calls: split each wrapper into a PURE `buildCmd(...) (argv,stdin,dir)` function + a thin `run` that execs it. Unit-test the command-builder (exact argv/flags/prompt) + the output parser — NEVER shell the real paid/slow CLI in CI (cost, auth, minutes). Guard any real-invocation smoke test behind an env flag (e.g. `BEEZLE_MODEL_SMOKE=1` + binary-on-PATH) the gate never sets, so it SKIPs. Bonus: strip `CLAUDECODE` from the child env (nested claude/codex/gemini SIGSEGV otherwise).
- Temporal: an ACTIVITY cannot start a child workflow (`workflow.ExecuteChildWorkflow` only works inside a workflow). To launch a sub-workflow from inside an activity (e.g. a DoTask handler that runs as an activity wanting to run a multi-step `BuildProgram` child), use the Temporal CLIENT (`client.ExecuteWorkflow`/`SignalWithStartWorkflow`) and (if you need the result) poll/await the run — or restructure so the parent workflow starts the child directly. Also: agentic build/fix loops (programmer→tester) MUST be bounded — a hard max-iterations cap AND early stop on repeated-identical failures — or they loop forever burning model calls.
- Latency budget tests in CI: NEVER assert a single timed sample against a hard budget (e.g. "first token < 1s") — a contended CI box blips one sample just over and the gate flakes (saw 1.0189s vs a 1s budget). Take the MIN of best-of-N warm samples and assert that: a single contended blip can't fail it, but a real regression pushes ALL samples over so it still fails. Pre-warm before timing so you measure the warm path the product actually guarantees.

## go:embed paths
- `//go:embed <path>` is always relative to the Go SOURCE FILE, not the module root. If `store.go` is in `pkg/store/` and you want to embed `migrations/001.sql`, the migrations directory must be `pkg/store/migrations/001.sql` — not `migrations/001.sql` at the module root. The compiler returns "no matching files found" with no helpful location hint.

## pgx/v5 pool + Postgres advisory locks
- Postgres advisory locks are SESSION-scoped: when a `pgxpool` connection goes back to the pool after `pg_try_advisory_lock`, the lock is orphaned on that backend and released whenever the pool closes that connection. For any lock that must last the process lifetime (singleton lock, long-lived serialization), use a dedicated `pgx.Connect()` connection — NOT the pool — and hold it until process exit. `pgxpool.Exec(ctx, "SELECT pg_try_advisory_lock($1)")` appears to work but silently releases on the next pool recycle.
- `go get github.com/jackc/pgx/v5` does NOT pull in `pgxpool`: `go get github.com/jackc/pgx/v5/pgxpool` is a separate command (adds `github.com/jackc/puddle/v2` to go.sum).

## Cross-compilation for mac mini (amd64)
- The mac mini in Darren's fleet (10.0.0.46) is Intel x86_64 (darwin/amd64) even though it's macOS. The laptop is arm64. Cross-compile: `GOOS=darwin GOARCH=amd64 go build ...`. The mac mini's Postgres only binds via TCP (no Unix socket at the default path); connect with `host=localhost port=5432` not the default socket.
