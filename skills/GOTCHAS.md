# Go Gotchas & Known Pitfalls

## Gio GUI Framework
- `material.NewTheme()` is expensive — allocates a shaper and font cache. In frame-loop code, create once at construction and store as a persistent field, never call per-frame.
- macOS frame handlers: NEVER call blocking operations (subprocess, cross-window `window.Option()`) from within a frame handler. The Cocoa main thread blocks waiting for the frame to complete → deadlock. Run all external/cross-window calls in goroutines.
- `key.Filter` with empty `Name` skips "SystemEvents" (Tab, Shift+Tab). To receive Tab in a widget, add explicit `key.Filter{Name: key.NameTab}` alongside the catch-all filter. Without it, Gio consumes Tab for focus navigation.
- `pointer.Filter` requires `ScrollY`/`ScrollX` bounds for scroll events. Default `{Min:0, Max:0}` silently rejects ALL scroll events. Always set bounds matching your scrollable content range.
- When debugging events not firing in a Gio app with multiple window types (ControlWindow + standalone TerminalWidget), instrument ALL window types — a missing log might mean the event goes to the OTHER window, not that it's missing entirely.

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
