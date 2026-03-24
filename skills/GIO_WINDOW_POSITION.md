# Gio UI — Window Position Persistence (macOS)

Gio has **no public API for window position**. `Config.Size` exists but `Config.Position` does not — a patch was proposed in 2022 and rejected by the maintainer.

**Workaround**: Use `AppKitViewEvent.View` (the native NSView handle) with CGo to call Cocoa APIs directly.

## Getting the native handle

```go
case app.AppKitViewEvent:
    if e.Valid() {
        viewHandle = e.View  // uintptr — CFTypeRef for NSView
    }
```

## CGo for position get/set

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

## Critical: Main Thread Dispatch (CRASH WITHOUT THIS)

AppKit calls MUST execute on the main thread. Calling `[window frame]` or `[window setFrame:]` from a Go goroutine causes `SIGTRAP: trace trap` crashes. Wrap window operations with `dispatch_sync`/`dispatch_async`:

```c
#include <dispatch/dispatch.h>

static void getWindowFrame(uintptr_t viewRef, CGFloat *ox, CGFloat *oy, CGFloat *ow, CGFloat *oh) {
    __block CGFloat bx=0, by=0, bw=0, bh=0;
    void (^work)(void) = ^{ /* read [window frame] */ };
    if ([NSThread isMainThread]) { work(); }
    else { dispatch_sync(dispatch_get_main_queue(), work); }
    *ox=bx; *oy=by; *ow=bw; *oh=bh;
}
```

- Use `dispatch_sync` for reads (getWindowFrame) — blocks until result available.
- Use `dispatch_async` for writes (setWindowFrame) — avoids deadlock during window creation.
- **Exception**: `[NSScreen screens]` is thread-safe for reading — no dispatch needed. Using `dispatch_sync` to main queue in tests (no run loop) deadlocks.

## Critical: No CGo or I/O in Gio Event Handlers

On macOS, the Cocoa main thread blocks while the event handler runs. CGo calls and file I/O in the handler cause "not responding" beach balls. Use a **background tracker goroutine**:

```go
// Background goroutine polls position every 100ms
func (w *Window) tracker() {
    ticker := time.NewTicker(100 * time.Millisecond)
    for { /* read frame via CGo, save to disk, call w.Invalidate() */ }
}
```

The event handler only reads a mutex-protected value — zero blocking operations.

## Library: daz-golang-gio/persist

All of the above is packaged in `github.com/darrenoakey/daz-golang-gio/persist`. One-line usage:

```go
w := persist.NewWindow("my-app", app.Title("My App"))
// Position and size automatically persisted. Use w.Frame() to read current state.
```

## Key gotchas

- **Coordinate system**: macOS origin is **bottom-left** of screen. `y` is distance from screen bottom to window bottom.
- **Screen coordinates vs Dp**: NSWindow frame is in screen points (not Gio `unit.Dp`). Save/restore using native frame values — do NOT mix with `app.Size()` which expects Dp.
- **Multi-monitor**: Validate saved position is still on-screen before restoring (monitors may change). Use `[NSScreen screens]` and `NSPointInRect`.
- **`AppKitViewEvent` timing**: Arrives after the window is created but before the first `FrameEvent`.
- **`AppKitViewEvent` is darwin-only**: Not visible on pkg.go.dev (docs generated for Linux). Type-assert `app.ViewEvent` to `app.AppKitViewEvent`.
- **Gio's `app.Size()` forces mode to `Windowed`**: If user was maximized, calling `Size()` un-maximizes.
- **Persistence**: Save frame to JSON atomically (tmp + rename). Use a background goroutine — never write from the event handler.
- Reference implementation: `~/src/go gui/window-memory/`
