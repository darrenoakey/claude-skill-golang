# Gio UI — Platform, Builds & Recipes

## Cross-Platform Builds (CGO Required)

Gio requires CGO. Pure Go builds (`CGO_ENABLED=0`) fail.

### Linux CI — System Headers Required

```bash
sudo apt-get install -y \
  gcc pkg-config \
  libwayland-dev libx11-dev libx11-xcb-dev \
  libxkbcommon-x11-dev libgles2-mesa-dev libegl1-mesa-dev \
  libffi-dev libxcursor-dev libvulkan-dev
```

### macOS — Works Out of the Box

macOS runners have Xcode CLT pre-installed. CGO including Objective-C (`#import <Cocoa/Cocoa.h>`) just works.

**Cross-compiling macOS arm64 → amd64**: Gio uses Metal on arm64, OpenGL on amd64. Without explicit arch flags, CGO generates arm64 GL bindings and fails with `undefined: Functions`:

```yaml
# GitHub Actions: build amd64 on macos-latest (Apple Silicon runner)
- name: Build amd64 (Intel)
  env:
    GOARCH: amd64
    CGO_ENABLED: "1"
    CC: clang -arch x86_64    # Required — without this: undefined: Functions
    CXX: clang++ -arch x86_64
  run: go build -o output/myapp-darwin-amd64 ./cmd/myapp
```

**Cross-compiling macOS from Linux**: Not practical when the project has Objective-C CGO. Use native macOS runners.

### Platform-Specific CGO Stubs

Files named `feature_darwin.go` are auto-excluded on non-macOS (Go's filename-based build constraints). If they define functions called from shared code (e.g. `main.go`), Linux/Windows builds fail with `undefined: funcName`.

Fix: create a stub for non-darwin platforms:

```go
// feature_other.go
//go:build !darwin

package main

// setDockIcon is a no-op on non-macOS platforms.
func setDockIcon() {}
```

The `_darwin.go` suffix handles macOS inclusion; `//go:build !darwin` covers everything else.

---

## Fonts

```go
// Embed fonts at compile time
//go:embed fonts/*.ttf
var fontFiles embed.FS

// Load with system fallback for Unicode support
collection, _ := gofont.Collection()
shaper := text.NewShaper(text.WithCollection(collection))
```

Font weight: `font.Bold` from `gioui.org/font` — **not** `text.Bold`.

---

## Material Design Icons

Use `golang.org/x/exp/shiny/materialdesign/icons` for standard Material Design icons in Gio. Each icon is a `[]byte` in IconVG format that `widget.NewIcon` decodes.

```go
import (
    "gioui.org/widget"
    "golang.org/x/exp/shiny/materialdesign/icons"
)

// Load once at construction — never per frame
refresh, _ := widget.NewIcon(icons.NavigationRefresh)
stop, _    := widget.NewIcon(icons.AVStop)
play, _    := widget.NewIcon(icons.AVPlayArrow)
```

Use with `material.IconButton` for buttons:
```go
btn := material.IconButton(th, &clickable, icon, "Description")
btn.Size = unit.Dp(18)
btn.Inset = layout.UniformInset(unit.Dp(6))
btn.Background = color.NRGBA{} // transparent — no filled circle
btn.Color = iconColor
btn.Layout(gtx)
```

Common icon paths: `icons.Navigation*`, `icons.AV*`, `icons.Content*`, `icons.Action*`, `icons.Communication*`.

---

## Single-Instance Daemon Pattern

For apps that should run as a single background process:

```go
// Acquire exclusive file lock (race-free via kernel)
lockPath := filepath.Join(socketDir, "daemon.lock")
lockFile, err := os.OpenFile(lockPath, os.O_CREATE|os.O_RDWR, 0644)
if err != nil { ... }
if err := syscall.Flock(int(lockFile.Fd()), syscall.LOCK_EX|syscall.LOCK_NB); err != nil {
    fmt.Fprintln(os.Stderr, "another instance is already running")
    os.Exit(0)
}
// CRITICAL: keep lockFile referenced or Go GC finalizes it (closes file = releases lock)
defer runtime.KeepAlive(lockFile)

// Daemonize: re-exec self with env var, detach from parent
cmd := exec.Command(os.Args[0])
cmd.Env = append(os.Environ(), "MYAPP_DAEMON=1")
cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true}  // detach from terminal
cmd.Start()  // don't Wait — let it run independently
```

**IPC**: Use Unix sockets (`net.Listen("unix", socketPath)`) for single-instance message passing. Set a deadline on accepted connections to prevent goroutine leaks:
```go
conn.SetDeadline(time.Now().Add(5 * time.Second))
```

---

## App Icons (Dock / Taskbar)

**Always generate icons with `~/bin/generate_image`** using `--transparent` for a proper transparent background PNG. Never use placeholder icons or skip icon generation.

```bash
# Generate a 512x512 transparent dock icon
~/bin/generate_image \
  --prompt "description of your app icon, centered on solid dark background" \
  --width 512 --height 512 \
  --output src/cmd/myapp/gui/icon.png \
  --transparent
```

Embed and set the dock icon on macOS via CGO (`NSApplication setApplicationIconImage:`). See the `icon_darwin.go` pattern with `//go:embed gui/icon.png`. Always provide a `_other.go` stub for non-darwin builds.

---

**For window position persistence on macOS**, read [GIO_WINDOW_POSITION.md](GIO_WINDOW_POSITION.md).
