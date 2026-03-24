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

## Key gotchas

- **Coordinate system**: macOS origin is **bottom-left** of screen. `y` is distance from screen bottom to window bottom.
- **Screen coordinates vs Dp**: NSWindow frame is in screen points (not Gio `unit.Dp`). Save/restore using native frame values — do NOT mix with `app.Size()` which expects Dp.
- **Multi-monitor**: Validate saved position is still on-screen before restoring (monitors may change). Use `[NSScreen screens]` and `NSPointInRect`.
- **`AppKitViewEvent` timing**: Arrives after the window is created but before the first `FrameEvent`. Restore position in the `AppKitViewEvent` handler.
- **`AppKitViewEvent` is darwin-only**: Not visible on pkg.go.dev (docs generated for Linux). Type-assert `app.ViewEvent` to `app.AppKitViewEvent`.
- **Gio's `app.Size()` forces mode to `Windowed`**: If user was maximized, calling `Size()` un-maximizes.
- **Persistence**: Save frame to JSON in `os.UserConfigDir()/appname/`. Write atomically (tmp + rename). Debounce saves — don't write every frame.
- Reference implementation: `~/src/go gui/window-memory/`
