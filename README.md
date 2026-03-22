# zui_qt

A native Qt/PySide6 retained-mode scene-graph toolkit for building zoomable 2D canvas applications with floating tool panels. Originally designed to demonstrate ZUI concepts, now a full application framework with layout, IPC, scripting, and 14 built-in widgets.

## Install and run

```bash
pip install PySide6
python example_mind_map_app.py    # mind map editor with scripting
python example_paint_app.py       # paint demo
python demo_bridge_signal.py      # IPC bridge demo (signal plotter)
python demo_layout.py             # layout system side-by-side comparison
```

## Architecture

The library is layered. Each layer only depends on the one below it.

```
┌─────────────────────────────────────────────────────────┐
│  Applications  (mind map, paint, signal plotter)        │
├─────────────────────────────────────────────────────────┤
│  Scripting     (exec Python from node notes)            │
│  Bridge        (IPC over TCP / Unix / stdio / process)  │
├─────────────────────────────────────────────────────────┤
│  Widgets       (window, button, label, slider, ...)     │
│  Layout        (vbox, hbox, spacer)                     │
│  Scroll        (scrollable viewport)                    │
│  TextArea      (multi-line text editor)                 │
├─────────────────────────────────────────────────────────┤
│  Core          (ZNode tree, ZEventDispatcher, ZContext)  │
│  Camera        (pan, zoom, coordinate transforms)       │
│  Canvas        (Qt bridge, 60fps paint loop)            │
│  Theme         (light/dark, colors, metrics)            │
│  Events        (ZPointer, ZKey wrappers)                │
│  Signals       (observer pattern, observable values)    │
└─────────────────────────────────────────────────────────┘
```

## Module reference

### Core — `zui_qt/core.py`

The scene graph and central facade.

**ZNode** is the base class for all visual elements. It has position (x, y), size (w, h), scale (sx, sy), visibility, and a tree of children. Override `draw_self(painter, ctx)` for custom rendering and `mouse_pressed(event)` for input handling. Hit testing, coordinate transforms (local to screen to global), and recursive drawing are built in.

**ZContainer** is a ZNode that clips its children to its bounds.

**ZEventDispatcher** routes pointer and keyboard events through the scene graph. It picks the deepest node under the cursor, bubbles press events up the parent chain until one consumes it, then captures drag/release to that node. It manages focus (one focused node receives key events) and a fallback handler for unhandled events (typically the canvas pan/zoom handler).

**ZContext** holds per-frame state: theme reference, camera reference, frame count, delta time, cursor blink clock, canvas size.

**Zui** is the facade that ties everything together. Create one per application. It provides factory methods for all widget types: `z.button(...)`, `z.vbox(...)`, `z.slider(...)`, and so on.

### Camera — `zui_qt/camera.py`

**ZCamera** is a 2D viewport with pan and zoom. It converts between screen and world coordinates, supports focal-point zoom (scroll wheel zooms toward cursor position), fit-to-rect, and painter transform application. The camera defines a rectangular viewport within the widget; everything outside is typically used for overlay panels.

Key methods: `screen_to_world` / `world_to_screen` for coordinate conversion, `zoom_at` for focal-point zoom, `fit_rect` for auto-framing a world rectangle, `apply_to_painter` / `restore_painter` for pushing and popping the transform, and `begin_pan` / `update_pan` / `end_pan` for drag panning.

### Events — `zui_qt/events.py`

**ZPointer** wraps mouse events with fields for x, y, button, buttons, modifiers, and delta_y. Helpers: `is_left()`, `is_middle()`, `is_right()`.

**ZKey** wraps keyboard events with key (int), text (str), and modifiers. Helpers: `is_command_key()` (Ctrl or Meta) and `is_shift()`.

**ZEventHandler** is the interface for canvas-level event handling (the fallback handler). Override `mouse_pressed`, `mouse_dragged`, `mouse_released`, `mouse_moved`, `mouse_wheel`, `key_pressed`, and `key_typed`.

### Canvas — `zui_qt/canvas.py`

**ZCanvas** is a QWidget subclass that bridges Qt events to the ZUI event system. It supports three rendering layers: a world renderer (drawn behind ZUI, typically under camera transform), the ZUI scene graph (drawn by the dispatcher), and an overlay renderer (drawn on top). It emits `hoveredNodeChanged` and `focusedNodeChanged` Qt signals. By default the canvas only repaints in response to input events. Call `canvas.set_continuous(True)` to enable a 60fps repaint timer for animations, cursor blink, or live data feeds.

### Theme — `zui_qt/theme.py`

**ZTheme** is a dataclass with all color tokens: bg, surface, surface_hover, surface_down, border, title_bg, title_fg, fg, fg_muted, accent, shadow, plus metrics for corner_radius, title_height, and font_size. Static factories `ZTheme.light()` and `ZTheme.dark()` provide complete presets.

**ZStyle** holds per-node style overrides for fill, stroke, and stroke_width.

### Layout — `zui_qt/layout.py`

Automatic layout containers that eliminate manual pixel positioning.

**ZVBox** stacks children top-to-bottom with configurable spacing and padding. Each child can have a `stretch` factor (like CSS flex-grow) to fill remaining space, or a `fixed_cross` to prevent cross-axis stretching. Pass `h=0` for auto-height that shrink-wraps content.

**ZHBox** stacks children left-to-right with the same API.

**ZSpacer** is an invisible node that absorbs space. Use `vbox.add(ZSpacer(), stretch=1)` to push subsequent children to the end.

```python
panel = z.vbox(0, 0, 200, spacing=6, padding=8)
panel.add(z.label(0, 0, 0, 18, "Title"))
panel.add(z.button(0, 0, 0, 24, "Click me", on_click))

row = z.hbox(0, 0, 0, 24, spacing=4)
row.add(z.button(0, 0, 0, 24, "OK", on_ok), stretch=1)
row.add(z.button(0, 0, 0, 24, "Cancel", on_cancel), stretch=1)
panel.add(row)

panel.add(z.spacer(), stretch=1)
panel.add(z.label(0, 0, 0, 14, "Footer"))
panel.do_layout()
```

Nested layouts auto-measure via `preferred_size()`. Invisible children are skipped.

### Scroll — `zui_qt/scroll.py`

**ZScrollView** clips and scrolls a single content node. It supports mouse wheel scrolling, a draggable scrollbar thumb, and programmatic scrolling via `scroll_to_reveal`, `scroll_to_top`, and `scroll_to_bottom`.

### Widgets — `zui_qt/widgets.py`

**ZWindow** is a draggable floating panel with a title bar containing a clipping content area. Drag the title bar to reposition.

**ZButton** is a click button with press/release visual feedback. Constructor: `ZButton(x, y, w, h, caption, on_click)`.

**ZLabel** is a static or dynamic text label. Pass a string or a `Callable[[], str]` for live-updating text.

**ZTextField** is a single-line text input with full editing: click-to-position caret, text selection (click-drag, Shift+arrows), clipboard (Ctrl+C/X/V), word navigation (Ctrl+Left/Right), and Ctrl+A select all. It uses a getter/setter pattern for two-way binding.

**ZStepper** is a numeric up/down control with plus and minus buttons.

**ZPaletteGrid** is a grid of selectable color swatches.

### Extended widgets — `zui_qt/widgets_ext.py`

**ZCheckbox** is a boolean toggle with a checkmark box and text label with an `on_changed(bool)` callback.

**ZToggle** is an iOS-style on/off switch with an optional label.

**ZSlider** is a horizontal slider with a draggable thumb and active track fill, configurable min/max range, and an `on_changed(float)` callback.

**ZDropdown** is a click-to-open option picker that shows a floating list below the button with hover highlighting. Press Escape to dismiss.

**ZProgressBar** is a horizontal progress indicator. Set `value` (0.0 to 1.0) for determinate mode, or `indeterminate = True` for an animated bouncing bar.

**ZSeparator** is a thin horizontal or vertical divider line.

**ZContextMenu** is a floating right-click menu with items, separators (`"---"`), disabled items, and hover highlighting. Call `show(x, y)` to display; items auto-dismiss on selection.

**ZTooltip** is a floating label for hover tooltips. Call `show_at(x, y, text)` and `hide()`.

### Text area — `zui_qt/textarea.py`

**ZTextArea** is a multi-line text editor widget with text selection (click-drag, Shift+arrows, Ctrl+A), clipboard (Ctrl+C/X/V), word navigation (Ctrl+Left/Right), a 200-step undo/redo stack (Ctrl+Z / Ctrl+Shift+Z), vertical scrolling with a scrollbar, an optional line number gutter, read-only mode, and placeholder text.

```python
editor = z.text_area(0, 0, 400, 300, "initial text",
                     on_changed=my_callback,
                     line_numbers=True,
                     placeholder="Type here...")
```

### Signals — `zui_qt/signals.py`

Lightweight observer system for reactive data flow.

**ZSignalMixin** mixes into any class to add `connect(signal, callback)`, `notify(signal, **kwargs)`, and `disconnect(signal)`. Callbacks receive keyword arguments.

**ZObservableValue** wraps a single value that fires `on_changed(value, old)` when set. Duplicate sets are suppressed.

**ZObservableList** wraps a list that fires `on_added(index, item)`, `on_removed(index, item)`, and `on_changed()` on mutations.

### Bridge — `zui_qt/bridge.py`

Bidirectional IPC between ZUI and external processes via a JSON-line protocol.

**ZBridge** is a message bus integrated with the Qt event loop. It supports four transports: `connect_tcp(host, port)` for TCP sockets, `connect_unix(path)` for Unix domain sockets, `connect_stdio()` for stdin/stdout when ZUI is a subprocess, and `connect_process(command)` to launch and pipe to a child process.

Outbound messages (ZUI to external):
```python
bridge.emit("compute.start", {"param": 42})
```

Inbound messages (external to ZUI):
```python
@bridge.on("result.ready")
def on_result(payload):
    my_label._text = payload["text"]
```

Triggers bind widget events to outbound messages automatically:
```python
bridge.trigger(my_button, "clicked", topic="compute.start")
```

Built-in commands let external processes manipulate the UI without custom handlers: `ui.set_property`, `ui.call`, `ui.theme`, `ui.status`.

The protocol format is JSON lines (newline-delimited):
```json
{"topic": "signal.data", "payload": {"t": 1.23, "y": 0.42}}
```

The external process can be written in any language that reads and writes JSON lines on stdin/stdout or a socket.

### Scripting — `zui_qt/scripting.py`

Execute Python code stored in mind-map node notes, turning the document into a live programming environment.

**ScriptEngine** takes a MindMapDocument and executes code from node notes. Three execution modes: `run_node(id)` executes one node, `run_subtree(id)` executes the node and descendants depth-first with parent before children, and `run_all()` resets the shared context and runs from root.

Code in a node's note receives these variables:
- `doc` — the MindMapDocument (can read/write any node)
- `node` — the current TopicNodeData
- `parent` — parent node data or None
- `children` — list of child node data
- `ctx` — shared dict for inter-node data flow (persists across nodes in a run)
- `bridge` — ZBridge instance if connected, for IPC
- `print()` — captured, output goes to node status

Set `__result__ = value` in a node's code to store a return value on the status object. Errors are caught with full tracebacks. Each node gets a `NodeExecStatus` with state (success, error, skipped, running), captured stdout, error message, duration, and result.

Visually, nodes with code show a ▶ run pill (bottom-right). After execution, a status dot appears: green for success, red for error, gray for skipped, amber for running. Collapsed subtrees are skipped during execution, so the collapse button doubles as an execution gate.

Example workflow: root node sets `ctx["db"] = connect(...)`, child node queries `ctx["db"].execute(...)`, grandchild formats and exports. The tree topology is the execution graph.

### Mind map — `zui_qt/mindmap.py`

The demonstration application's data and rendering layer.

**TopicNodeData** is the data model for a topic node with id, title, position, size, color, note (stores text or code), parent_id, children (ordered list of child IDs), collapsed flag, and branch_side.

**MindMapDocument** manages a tree of TopicNodeData with JSON serialization. Operations include add/remove/reparent nodes, set properties, compute visible nodes respecting collapse, and calculate bounding rectangles.

**MindMapLayoutEngine** positions nodes in a balanced left/right tree layout around the root.

**MindMapScene** handles rendering: Bézier curve connections, rounded-rect nodes with shadows, selection/hover highlights, collapse pills, run pills, and execution status dots. It also handles hit testing for node picking.

**CommandStack** provides snapshot-based undo/redo by storing full document JSON before and after each mutation.

### Inspectors — `zui_qt/inspector.py`, `zui_qt/mindmap_inspector.py`

Optional dock widgets that display scene graph structure (SceneInspectorDock) or mind map document outline (MindMapInspectorDock) as HTML. They prefer QWebEngineView when available and fall back to QTextBrowser.

## File map

```
zui_qt/
├── __init__.py           Public exports
├── core.py               ZNode, ZContainer, ZEventDispatcher, ZContext, Zui
├── camera.py             ZCamera (pan/zoom viewport)
├── events.py             ZPointer, ZKey, ZEventHandler
├── canvas.py             ZCanvas (Qt widget bridge)
├── theme.py              ZTheme, ZStyle
├── layout.py             ZVBox, ZHBox, ZSpacer
├── scroll.py             ZScrollView
├── widgets.py            ZWindow, ZButton, ZLabel, ZTextField, ZStepper, ZPaletteGrid
├── widgets_ext.py        ZCheckbox, ZToggle, ZSlider, ZDropdown, ZProgressBar,
│                         ZSeparator, ZContextMenu, ZTooltip
├── textarea.py           ZTextArea
├── signals.py            ZSignalMixin, ZObservableValue, ZObservableList
├── bridge.py             ZBridge (IPC)
├── scripting.py          ScriptEngine, ExecState, NodeExecStatus
├── mindmap.py            MindMapDocument, MindMapScene, MindMapLayoutEngine, CommandStack
├── inspector.py          SceneInspectorDock
└── mindmap_inspector.py  MindMapInspectorDock

example_mind_map_app.py   Full mind-map editor with scripting
example_paint_app.py      Paint demo
demo_bridge_signal.py     IPC bridge demo (signal plotter)
demo_layout.py            Layout system comparison
worker_signal_gen.py      External worker for bridge demo
test_layout.py            Layout system unit tests
```

## Mind map keyboard shortcuts

| Key | Action |
|-----|--------|
| Tab | Add child node |
| Shift+Enter | Add sibling node |
| F2 or Enter | Rename selected node inline |
| Delete or Backspace | Delete selected node |
| Space | Collapse or expand children |
| N | Edit note |
| R | Run node code |
| Shift+R | Run subtree |
| Ctrl+R | Run all |
| Arrow keys | Navigate selection |
| \[ and \] | Promote and demote |
| 0 | Fit document in view |
| + and - | Zoom in and out |
| Ctrl+Z and Ctrl+Y | Undo and redo |
| Ctrl+S | Save JSON |
| Ctrl+O | Open JSON |
| Ctrl+N | New document |
| Ctrl+L | Auto layout |
| Middle-click drag | Pan canvas |
| Scroll wheel | Zoom |
| Shift+drag | Reparent by dropping onto target node |
| Click ▶ pill | Run single node |

## Design notes

The rendering core is entirely native QPainter. There is no HTML, WebGL, or OpenGL involved in the main rendering path. The inspector docks are the only HTML-oriented pieces, and they are optional.

The mind map uses a separate document model (MindMapDocument) so application state is never owned by the visual tree. The ZUI scene graph handles rendering and interaction while the document model handles data, serialization, and undo. This separation is the recommended pattern for any application built on the library.

The IPC bridge uses JSON lines over pipes or sockets, making the external process language-agnostic. The ZUI side runs all I/O through Qt's event loop so the GUI never blocks.

Node scripting executes unsandboxed Python. This is a power-user feature for desktop tools where the user authors the code. Do not run untrusted documents with scripting enabled.

## Stats

- 17 library modules, ~5,200 lines
- 14 widget types
- 6 demo/test files, ~2,200 lines
- Total: ~7,400 lines of Python

## AI-assistance disclosure

This project was created by a human author with significant assistance from
an AI coding assistant (OpenAI’s ChatGPT / GPT-5.1 Thinking).

The AI was used for:

- Brainstorming the architecture and feature set.
- Iterating on the interpreter logic and JSON runtime design.
- Helping refine the GUI bridge and example scripts.
- Producing and polishing documentation like this README.

All code and decisions were reviewed and accepted by a human before being committed.

