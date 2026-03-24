# Gio UI — Layout Patterns

## Anchored Header (logo-left / search-center / status-right)

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

## Image with Aspect Ratio

```go
// widget.Contain preserves aspect ratio within bounding box
gtx.Constraints.Max = image.Pt(maxW, maxH)
widget.Image{Src: imageOp, Fit: widget.Contain}.Layout(gtx)
```

## Right-Click Detection

```go
case pointer.Event:
    if e.Kind == pointer.Press &&
        (e.Buttons.Contain(pointer.ButtonSecondary) ||
         (e.Buttons.Contain(pointer.ButtonPrimary) && e.Modifiers.Contain(key.ModCtrl))) {
        // Show context menu
    }
```

## Context Menu Component

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
