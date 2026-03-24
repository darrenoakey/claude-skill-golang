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

## Sub-files

- **[GIO_GOTCHAS.md](GIO_GOTCHAS.md)**: Critical gotchas (widget persistence, theme allocation, macOS deadlocks, Tab events, scroll bounds, pointer filters, focus, clipboard) and keyboard input handling. **Read this first.**
- **[GIO_PATTERNS.md](GIO_PATTERNS.md)**: Layout patterns (anchored header, images, right-click, context menu component).
- **[GIO_PLATFORM.md](GIO_PLATFORM.md)**: Cross-platform CGO builds, platform stubs, daemon pattern, app icons, fonts, Material Design icons.
- **[GIO_WINDOW_POSITION.md](GIO_WINDOW_POSITION.md)**: Window position persistence on macOS (AppKitViewEvent, CGo Cocoa APIs, coordinate gotchas).
