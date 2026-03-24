# Gio UI Framework

Use `gioui.org` for Go desktop GUI applications. Gio is an **immediate-mode** UI library — the entire frame layout is recomputed on every repaint. There is no persistent widget tree.

## Core Concepts

**Immediate-mode loop**: The frame handler runs on every repaint. Layout is not stored — it is re-executed each frame from scratch.

**Operations (Ops)**: Drawing and event registration happen by appending to an `op.Ops` buffer. The buffer is reset each frame.

**Units**: Use `unit.Dp` for density-independent pixels. Never use raw pixel integers.

**Window creation**:
```go
// CORRECT: new() then Option() separately
win := new(app.Window)
win.Option(app.Title("My App"), app.Size(unit.Dp(800), unit.Dp(600)))

// WRONG: app.NewWindow() does not exist in Gio
```

## Key Packages

| Package | Purpose |
|---------|---------|
| `gioui.org/app` | Window management, event loop |
| `gioui.org/layout` | Flex, Stack, Inset, List |
| `gioui.org/widget` | Clickable, Editor, Image, Label |
| `gioui.org/widget/material` | Material Design theme and components |
| `gioui.org/op` | Op buffer, offset, clip, transform |
| `gioui.org/io/key` | Keyboard events and filters |
| `gioui.org/io/pointer` | Pointer/mouse events and filters |
| `gioui.org/io/clipboard` | Clipboard read/write |
| `gioui.org/text` | Text shaping/rendering |
| `gioui.org/font` | Font loading and embedding |
| `gioui.org/unit` | dp, sp, px unit types |

---

## Critical Gotchas (Read These First)

### 1. Widget State Must Persist Across Frames

Gio routes events by matching the registered target object identity. Creating new widget objects each frame means events are never delivered.

```go
// WRONG: new Clickable each frame — clicks never register
func (w *Window) layout(gtx layout.Context) layout.Dimensions {
    btn := widget.Clickable{}                          // NEW each frame = broken
    return material.Button(th, &btn, "Click").Layout(gtx)
}

// CORRECT: persistent field on the struct
type Window struct {
    btn widget.Clickable
}
func (w *Window) layout(gtx layout.Context) layout.Dimensions {
    return material.Button(th, &w.btn, "Click").Layout(gtx)
}
```

For dynamic lists (tabs, sessions), use a **persistent map keyed by stable ID**:
```go
type Window struct {
    tabStates map[string]*widget.Clickable  // key: session name or ID
}
```
Clean up stale entries when items are removed, otherwise memory leaks.

### 2. Theme: Create Once — Never Per Frame

`material.NewTheme()` allocates a shaper and font cache. Calling it in a frame handler causes massive per-frame allocations.

```go
// WRONG: allocates on every frame
func (w *Window) layout(gtx layout.Context) layout.Dimensions {
    th := material.NewTheme()  // DO NOT DO THIS
    ...
}

// CORRECT: created once at construction, stored as field
type Window struct {
    theme *material.Theme
}
func NewWindow() *Window {
    return &Window{theme: material.NewTheme()}
}
```

### 3. macOS Frame Handler Deadlock (CRITICAL)

On macOS, the Cocoa main thread blocks while the Gio frame handler runs. Any operation that tries to dispatch back to the main thread from within the frame handler causes a deadlock (app beach-balls forever).

**Never do inside a frame handler:**
- `exec.Command(...).Run()` or any subprocess call
- `window.Option()` on a different Gio window
- Any blocking I/O or channel receive

**Always use a goroutine**, then call `window.Invalidate()` when done:

```go
// WRONG: deadlock on macOS
func (w *Window) layout(gtx layout.Context) layout.Dimensions {
    if w.btn.Clicked(gtx) {
        exec.Command("open", url).Run()  // DEADLOCK on macOS
    }
}

// CORRECT: goroutine for anything blocking or cross-window
func (w *Window) layout(gtx layout.Context) layout.Dimensions {
    if w.btn.Clicked(gtx) {
        go func() {
            exec.Command("open", url).Run()
            w.win.Invalidate()
        }()
    }
}
```

### 4. Tab Key is a SystemEvent

Gio intercepts Tab for focus navigation and delivers it as a `key.Event` with `Modifiers` set to `key.ModFunction`. A catch-all `key.Filter` (empty `Name`) skips `SystemEvent`s entirely — Tab is consumed and never reaches your handler.

```go
// WRONG: Tab silently consumed by Gio focus navigation
key.Filter{Name: "", Optional: key.ModShift | key.ModCtrl}

// CORRECT: explicit Tab filter alongside the catch-all
// Register BOTH filters:
key.Filter{Name: key.NameTab}                           // explicit Tab
key.Filter{Name: "", Optional: key.ModShift | key.ModCtrl}  // everything else
```

### 5. Scroll Events Require Bounds

`pointer.Filter` silently rejects scroll events when `ScrollY` or `ScrollX` bounds are `{Min: 0, Max: 0}` (the zero value). Always set bounds:

```go
// WRONG: scroll events silently dropped
pointer.InputOp{Tag: w, Types: pointer.Scroll}

// CORRECT: set bounds matching scrollable content range
pointer.InputOp{
    Tag:     w,
    Types:   pointer.Scroll,
    ScrollY: pointer.ScrollRange{Min: -1e6, Max: 1e6},
}
```

### 6. All Pointer Types in One Filter

Registering separate `pointer.InputOp` calls does not combine them — only the last one wins. Always put all required event types in a single call:

```go
// WRONG: only the last registration wins — Press never arrives
pointer.InputOp{Tag: w, Types: pointer.Press}
pointer.InputOp{Tag: w, Types: pointer.Release}

// CORRECT: combine into one
pointer.InputOp{
    Tag:   w,
    Types: pointer.Press | pointer.Drag | pointer.Release | pointer.Scroll,
}
```

### 7. Keyboard Focus Must Be Explicitly Requested

When switching keyboard input between handlers (e.g. rename input → terminal), explicitly request focus:

```go
gtx.Execute(key.FocusCmd{Tag: targetWidget})
```

Without this, `key.EditEvent` (typed characters) won't be delivered to the new handler even if the old one stops consuming events.

**Focus fight**: A widget with `skipKeyboard=true` (parent handles keyboard) must NOT call `gtx.Execute(key.FocusCmd{Tag: w})` on `pointer.Press`. Doing so steals focus from the parent's keyboard handler, breaking clipboard (Cmd+C/V) and all key input.

### 8. Clipboard MIME Type

Gio uses `"application/text"` — **not** `"text/plain"`. Using the wrong type causes silent failure — `transfer.DataEvent` never arrives.

```go
// Write (copy)
clipboard.WriteCmd{
    Type: "application/text",  // NOT "text/plain"
    Data: io.NopCloser(strings.NewReader(text)),
}

// Read (paste) — on macOS, bypass Gio clipboard entirely:
// Gio's clipboard only works for Gio-internal data (MIME mismatch for cross-app paste)
out, err := exec.Command("pbpaste").Output()  // macOS: run in goroutine
```

---

## Keyboard Input

Handle **both** `key.Event` and `key.EditEvent`. `EditEvent` carries properly composed text (handles dead keys, IME, shift). `key.Event` carries raw key names.

```go
for {
    e, ok := gtx.Event(
        key.Filter{Name: key.NameTab},
        key.Filter{Name: "", Optional: key.ModShift | key.ModCtrl | key.ModCommand},
    )
    if !ok {
        break
    }
    switch e := e.(type) {
    case key.Event:
        if e.State != key.Press {
            continue
        }
        // e.Name is UPPERCASE: "A", "Return", "Escape", "Tab"
        // Must handle e.Modifiers for proper case (Shift+"A" = "A", no Shift = "a")
    case key.EditEvent:
        // e.Text is the correctly composed character — use this for text input
    }
}
```

---

## Layout Patterns

### Anchored Header (logo-left / search-center / status-right)

Use `op.Offset()` with calculated pixel coordinates. **Do not use** Flex+spacers for this — Flex spacers do not reliably center when siblings have unequal widths.

```go
// Calculate positions
logoW := 120
searchW := 300
statusW := 150
totalW := gtx.Constraints.Max.X

// Logo: left-aligned
op.Offset(image.Pt(padding, (headerH-logoH)/2)).Add(gtx.Ops)
drawLogo(gtx)

// Search: centered
op.Offset(image.Pt((totalW-searchW)/2, (headerH-searchH)/2)).Add(gtx.Ops)
drawSearch(gtx)

// Status: right-aligned
op.Offset(image.Pt(totalW-statusW-padding, (headerH-statusH)/2)).Add(gtx.Ops)
drawStatus(gtx)
```

### Image with Aspect Ratio

```go
// widget.Contain preserves aspect ratio within bounding box
gtx.Constraints.Max = image.Pt(maxW, maxH)
widget.Image{Src: imageOp, Fit: widget.Contain}.Layout(gtx)
```

### Right-Click Detection

```go
case pointer.Event:
    if e.Kind == pointer.Press &&
        (e.Buttons.Contain(pointer.ButtonSecondary) ||
         (e.Buttons.Contain(pointer.ButtonPrimary) && e.Modifiers.Contain(key.ModCtrl))) {
        // Show context menu
    }
```

### Context Menu Component

A reusable floating context menu with hover highlighting, click-outside dismiss, and per-item actions. Key patterns:

**Structure**: Persistent struct with stable pointer tags for event routing.

```go
type ContextMenu struct {
    visible    bool
    showFrame  bool        // prevents dismiss on the frame Show() was called
    pos        image.Point // raw cursor position at show time
    items      []MenuItem
    itemTags   []*bool     // stable pointers — use []*bool not []bool
    bgTag      bool        // background dismiss handler
    hoverIdx   int         // -1 = none
    targetPID  int32       // caller-specific payload
    targetName string
}
```

**PassOp on the dismiss overlay (CRITICAL)**: The dismiss area covers the full window. Without `pointer.PassOp`, it is the topmost handler in the op tree and **steals all pointer events** from handlers underneath (row click handlers, buttons, etc.). This causes the "click to dismiss, next click does nothing" bug — the underlying handler never receives the event.

```go
// WRONG: bg overlay blocks all events to handlers underneath
bgArea := clip.Rect{Max: gtx.Constraints.Max}.Push(gtx.Ops)
event.Op(gtx.Ops, &m.bgTag)
bgArea.Pop()

// CORRECT: PassOp lets events pass through to underlying handlers
passStack := pointer.PassOp{}.Push(gtx.Ops)
bgArea := clip.Rect{Max: gtx.Constraints.Max}.Push(gtx.Ops)
event.Op(gtx.Ops, &m.bgTag)
bgArea.Pop()
passStack.Pop()
```

**Stale event drain on Show()**: When `Show()` is called (e.g. by a row right-click handler), the bg overlay from the previous frame has the same right-click event queued. Without draining, `Layout()` would immediately dismiss the menu it just opened. Set a `showFrame` flag in `Show()` and drain stale events at the start of `Layout()` when the flag is set.

```go
func (m *ContextMenu) Show(pos image.Point, ...) {
    m.visible = true
    m.showFrame = true  // signal to drain stale events this frame
    m.hoverIdx = -1
    // ...
}

func (m *ContextMenu) Layout(gtx layout.Context, th *material.Theme) menuResult {
    if !m.visible {
        m.drainEvents(gtx)  // always drain when hidden
        return menuResult{}
    }
    if m.showFrame {
        m.showFrame = false
        m.drainEvents(gtx)  // discard stale bg events from previous frame
    }
    // ... register bg overlay with PassOp, check dismiss, render items
}
```

**Hover highlighting**: Register `pointer.Enter | pointer.Leave | pointer.Press` on each item's clip area. Track `hoverIdx` and paint a highlight background before the label.

```go
// Register events
itemArea := clip.Rect{Max: image.Pt(itemW, itemH)}.Push(gtx.Ops)
event.Op(gtx.Ops, tag)
itemArea.Pop()

// Process events
for {
    ev, ok := gtx.Event(pointer.Filter{
        Target: tag,
        Kinds:  pointer.Press | pointer.Enter | pointer.Leave,
    })
    if !ok { break }
    if e, ok := ev.(pointer.Event); ok {
        switch e.Kind {
        case pointer.Enter:  m.hoverIdx = i
        case pointer.Leave:  if m.hoverIdx == i { m.hoverIdx = -1 }
        case pointer.Press:  // trigger action, dismiss
        }
    }
}

// Draw hover background before label
if m.hoverIdx == i {
    paint.FillShape(gtx.Ops, hoverColor, clip.Rect{Max: image.Pt(itemW, itemH)}.Op())
}
```

**Position offset**: Native context menus appear with the first item vertically centered at the cursor, shifted slightly left. Apply an offset before clamping to window bounds:

```go
displayPos := clampPosition(
    image.Pt(m.pos.X-8, m.pos.Y-itemH/2-padTop),
    menuW, totalH, gtx.Constraints.Max.X, gtx.Constraints.Max.Y,
)
```

**Execution order**: Row handlers run in `layoutTable` (before `menu.Layout`). Right-click calls `Show()` which sets `showFrame=true`. Then `menu.Layout` runs, sees `showFrame`, drains stale bg events, and renders the menu. The PassOp ensures both the bg overlay and row handlers receive future events.

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

## Cross-Platform Builds for Gio (CGO Required)

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

---

## Window Position Persistence (macOS)

Gio has **no public API for window position**. `Config.Size` exists but `Config.Position` does not — a patch was proposed in 2022 and rejected by the maintainer.

**Workaround**: Use `AppKitViewEvent.View` (the native NSView handle) with CGo to call Cocoa APIs directly.

### Getting the native handle

```go
case app.AppKitViewEvent:
    if e.Valid() {
        viewHandle = e.View  // uintptr — CFTypeRef for NSView
    }
```

### CGo for position get/set

Use `uintptr_t` in C signatures to avoid `go vet` "misuse of unsafe.Pointer" warnings:

```c
// #cgo CFLAGS: -x objective-c -fobjc-arc
// #cgo LDFLAGS: -framework AppKit
#import <AppKit/AppKit.h>
#include <stdint.h>

static void getWindowFrame(uintptr_t viewRef, CGFloat *x, CGFloat *y, CGFloat *w, CGFloat *h) {
    @autoreleasepool {
        NSView *view = (__bridge NSView *)(void *)viewRef;
        NSWindow *window = view.window;
        if (!window) { *x=0; *y=0; *w=0; *h=0; return; }
        NSRect frame = [window frame];
        *x = frame.origin.x; *y = frame.origin.y;
        *w = frame.size.width; *h = frame.size.height;
    }
}

static void setWindowFrame(uintptr_t viewRef, CGFloat x, CGFloat y, CGFloat w, CGFloat h) {
    @autoreleasepool {
        NSView *view = (__bridge NSView *)(void *)viewRef;
        NSWindow *window = view.window;
        if (!window) return;
        [window setFrame:NSMakeRect(x, y, w, h) display:YES];
    }
}
```

Go side: `C.getWindowFrame(C.uintptr_t(viewHandle), &cx, &cy, &cw, &ch)`

### Key gotchas

- **Coordinate system**: macOS origin is **bottom-left** of screen. `y` is distance from screen bottom to window bottom.
- **Screen coordinates vs Dp**: NSWindow frame is in screen points (not Gio `unit.Dp`). Save/restore using native frame values — do NOT mix with `app.Size()` which expects Dp.
- **Multi-monitor**: Validate saved position is still on-screen before restoring (monitors may change). Use `[NSScreen screens]` and `NSPointInRect`.
- **`AppKitViewEvent` timing**: Arrives after the window is created but before the first `FrameEvent`. Restore position in the `AppKitViewEvent` handler.
- **`AppKitViewEvent` is darwin-only**: Not visible on pkg.go.dev (docs generated for Linux). Type-assert `app.ViewEvent` to `app.AppKitViewEvent`.
- **Gio's `app.Size()` forces mode to `Windowed`**: If user was maximized, calling `Size()` un-maximizes.
- **Persistence**: Save frame to JSON in `os.UserConfigDir()/appname/`. Write atomically (tmp + rename). Debounce saves — don't write every frame.
- Reference implementation: `~/src/go gui/window-memory/`

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
