# Go Gotchas & Known Pitfalls

## Gio GUI Framework
- `material.NewTheme()` is expensive — allocates a shaper and font cache. In frame-loop code, create once at construction and store as a persistent field, never call per-frame.
- macOS frame handlers: NEVER call blocking operations (subprocess, cross-window `window.Option()`) from within a frame handler. The Cocoa main thread blocks waiting for the frame to complete → deadlock. Run all external/cross-window calls in goroutines.
- `key.Filter` with empty `Name` skips "SystemEvents" (Tab, Shift+Tab). To receive Tab in a widget, add explicit `key.Filter{Name: key.NameTab}` alongside the catch-all filter. Without it, Gio consumes Tab for focus navigation.
- `pointer.Filter` requires `ScrollY`/`ScrollX` bounds for scroll events. Default `{Min:0, Max:0}` silently rejects ALL scroll events. Always set bounds matching your scrollable content range.
- When debugging events not firing in a Gio app with multiple window types (ControlWindow + standalone TerminalWidget), instrument ALL window types — a missing log might mean the event goes to the OTHER window, not that it's missing entirely.
- `SetAnimating()` + nil/broken CVDisplayLink: launch a 60Hz `time.Ticker` fallback goroutine. Do NOT add `setNeedsDisplay` inside `SetAnimating` — creates a busy-loop.
- Hardware sync handles can be created (non-nil) but fail to deliver callbacks. Pair hardware sync with a software fallback — NOT by adding side-effects to hot-path methods.
- `clipboard.WriteCmd` is deferred to the next frame. On macOS with broken display link, use `exec.Command("pbcopy")`/`exec.Command("pbpaste")` instead.
- Mouse selection auto-copy: only call pbcopy on `pointer.Release` when selection has real extent (start ≠ end). A single click overwrites the clipboard.

## CGO & Cross-compilation
- `go:embed` embeds files into the binary at compile time. Changes to embedded files (HTML/CSS/JS in dashboards) require rebuilding the binary and restarting the process to take effect. The embedded files are NOT read from disk at runtime.
- Platform-specific CGO stubs: if `_darwin.go` defines a function called from non-platform-specific code (e.g. `main.go`), Linux builds fail with `undefined: FuncName`. Fix: add `icon_other.go` with `//go:build !darwin` containing a no-op stub. The `_darwin.go` filename suffix auto-excludes it; the stub covers all other platforms.
- CGO cross-compilation macOS arm64→amd64: Set `CC: clang -arch x86_64` and `CXX: clang++ -arch x86_64` alongside `GOARCH: amd64` in the CI env. Without explicit arch flags, CGO may pick wrong GL bindings and fail with `undefined: Functions` (Gio v0.7.1 uses Metal on arm64, OpenGL on amd64).
- Go+Gio GUI apps on Linux CI runners require CGO system headers: `sudo apt-get install -y gcc pkg-config libwayland-dev libx11-dev libx11-xcb-dev libxkbcommon-x11-dev libgles2-mesa-dev libegl1-mesa-dev libffi-dev libxcursor-dev libvulkan-dev`. macOS runners work out of the box (Xcode CLT handles ObjC CGO).

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

## Lambda / Deployment
- Compile with `-ldflags="-s -w"` to strip debug symbols. Lambda's 6MB limit applies to the **compressed** zip, not raw binary.

## Runtime
- Long-running daemons: set `debug.SetMemoryLimit(N)` at startup. Without it, Go GC only triggers when heap doubles — transient bursts can balloon to OOM. Set to ~2.5x expected steady-state RSS.
- `macOS filepath.EvalSymlinks` on non-existent paths: `/var/folders` is a symlink to `/private/var/folders`. Walk the ancestor chain to find the first existing directory, resolve it, then reattach remaining components.
- gopsutil on macOS: `p.Name()` returns "Python" (capital P). Use `strings.ToLower` for interpreter matching. Use `p.Cwd()` for meaningful process identification.
- Unrecovered panic in ANY goroutine kills the entire process. Always add `defer func() { if r := recover(); r != nil { log(r) } }()` in fire-and-forget goroutines.

## Templates / Web
- `html/template` with `embed.FS`: use `fs.Sub(staticFSRaw, "static")` before `http.FileServer`. Multiple template files defining `{{define "content"}}` conflict — parse layout once, then `Clone()` per page.
- SSE endpoints (`text/event-stream`) never close — Playwright's `networkidle` hangs forever. Delay `new EventSource()` in JS so the page's `load` event fires first.

## SQL / database/sql
- `json.RawMessage` (`[]byte`) cannot scan NULL from PostgreSQL — use `sql.NullString` instead when a LEFT JOIN may produce NULL columns. Convert with `json.RawMessage(nullStr.String)` after checking `.Valid`.
- **SQLite/PostgreSQL dual-driver pattern**: Write all SQL with `?` placeholders, convert to `$1, $2, ...` for PostgreSQL via a helper. Both `modernc.org/sqlite` (driver name `"sqlite"`, pure Go, no CGO) and `github.com/lib/pq` (driver name `"postgres"`) support `ON CONFLICT ... DO UPDATE SET` for upserts.
- For SQLite via `database/sql`, call `db.SetMaxOpenConns(1)` to avoid concurrent write issues.
- **Prepared statements for remote PostgreSQL**: Query planning adds ~25-30ms overhead per call. For hot-path queries called 1000+ times, use `db.Prepare()` at init and cache the `*sql.Stmt` on the Store struct. Cuts per-call latency in half for remote databases.
- **Batch DB writes into transactions**: 100 operations in one `BeginTx/Commit` is ~100x faster than 100 auto-commit calls (1 network round-trip + 1 fsync vs 100 of each). Use `WriteItemResult`-style methods that bundle value insert + state update + execution record + event in a single tx.
- **CTE planning overhead**: Complex CTEs with UPDATE...RETURNING pay ~30ms PostgreSQL planning time per call. Prefer simple UPDATE...FROM + separate INSERT over a CTE when called in tight loops.

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
- Query parameters with spaces MUST be URL-encoded with `url.QueryEscape()`. Unencoded spaces cause HTTP 400 `failedPrecondition`.
