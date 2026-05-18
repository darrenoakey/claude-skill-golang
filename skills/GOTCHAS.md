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
- **SQLite/PostgreSQL dual-driver pattern**: Write all SQL with `?` placeholders, convert to `$1, $2, ...` for PostgreSQL via a helper. Both `modernc.org/sqlite` (driver name `"sqlite"`, pure Go, no CGO) and `github.com/lib/pq` (driver name `"postgres"`) support `ON CONFLICT ... DO UPDATE SET` for upserts.
- For SQLite via `database/sql`, call `db.SetMaxOpenConns(1)` to avoid concurrent write issues.
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
