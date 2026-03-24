# Gio UI — Critical Gotchas & Keyboard Input

**Read these before writing any Gio code.**

## 1. Widget State Must Persist Across Frames

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

## 2. Theme: Create Once — Never Per Frame

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

## 3. macOS Frame Handler Deadlock (CRITICAL)

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

## 4. Tab Key is a SystemEvent

Gio intercepts Tab for focus navigation and delivers it as a `key.Event` with `Modifiers` set to `key.ModFunction`. A catch-all `key.Filter` (empty `Name`) skips `SystemEvent`s entirely — Tab is consumed and never reaches your handler.

```go
// WRONG: Tab silently consumed by Gio focus navigation
key.Filter{Name: "", Optional: key.ModShift | key.ModCtrl}

// CORRECT: explicit Tab filter alongside the catch-all
// Register BOTH filters:
key.Filter{Name: key.NameTab}                           // explicit Tab
key.Filter{Name: "", Optional: key.ModShift | key.ModCtrl}  // everything else
```

## 5. Scroll Events Require Bounds

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

## 6. All Pointer Types in One Filter

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

## 7. Keyboard Focus Must Be Explicitly Requested

When switching keyboard input between handlers (e.g. rename input → terminal), explicitly request focus:

```go
gtx.Execute(key.FocusCmd{Tag: targetWidget})
```

Without this, `key.EditEvent` (typed characters) won't be delivered to the new handler even if the old one stops consuming events.

**Focus fight**: A widget with `skipKeyboard=true` (parent handles keyboard) must NOT call `gtx.Execute(key.FocusCmd{Tag: w})` on `pointer.Press`. Doing so steals focus from the parent's keyboard handler, breaking clipboard (Cmd+C/V) and all key input.

## 8. Clipboard MIME Type

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
