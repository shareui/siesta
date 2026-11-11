# Runtime API and Architecture of siesta

The `siesta_core` module is the high-performance runtime engine for terminal user interfaces written in Yukino. While the procedural macro `#[siesta::res]` compiles `.spes` markup into native classes, `siesta_core` handles all low-level responsibilities: raw terminal I/O, double buffering, flexbox constraint layout calculation, cyclic focus traversal, mouse dispatching, and zero-leak memory management.

---

## 1. Importing the Runtime: load siesta

All public structures, classes, layout algorithms, and widgets are imported through the unified `siesta` module:

```yukino
load siesta: Cell, init_cell, cells_equal, styles_equal,
             TerminalBuffer, DoubleBuffer, DiffSpan, compute_diff_spans, flush_diff_spans,
             Rect, init_rect, rect_contains, rect_inner, rect_intersect,
             LayoutContext, Screen, Dialog, SiestaApp, detect_is_dark_mode,
             Constraint, constraint_length, constraint_percentage, constraint_ratio,
             constraint_min, constraint_max, constraint_fill, solve_constraints,
             layout_split_horizontal, layout_split_vertical, layout_row, layout_col, layout_grid,
             FocusNode, FocusManager, MouseTarget, MouseDispatchResult, MouseRouter, ModalEntry, ModalManager,
             Transition, Pulse, Slide, AnimationTicker, sample_easing,
             linear, ease_in_quad, ease_out_quad, ease_in_out_quad, ease_in_cubic,
             ease_out_cubic, ease_in_out_cubic, bounce_out, bounce_in, bounce_in_out, elastic_out,
             TextWidget, BlockWidget, ParagraphWidget, ButtonWidget, InputWidget, TextAreaWidget,
             CheckboxWidget, RadioGroupWidget, ListWidget, TableWidget, TableColumn, TableRow,
             ProgressWidget, GaugeWidget, LineGaugeWidget, SparklineWidget, BarChartWidget,
             ScrollbarWidget, TabsWidget, TreeWidget, TreeItem, init_tree_item, CanvasWidget,
             BORDER_NONE, BORDER_PLAIN, BORDER_ROUNDED, BORDER_DOUBLE, BORDER_THICK, BORDER_ASCII,
             ALIGN_LEFT, ALIGN_CENTER, ALIGN_RIGHT, WRAP_NONE, WRAP_CHAR, WRAP_WORD,
             Logger, log_init, log_debug
```

---

## 2. Fundamental Data Primitives: Cell and Rect

### 2.1 Terminal Grid Cell: Cell
Every point on the terminal screen is represented by a `Cell` struct. It is an indivisible graphical quantum storing one UTF-8 grapheme alongside its visual styling attributes:

```yukino
public struct Cell {
    ch: string
    fg: string
    bg: string
    bold: bool
    dim: bool
    italic: bool
    underline: bool
    reverse: bool
}
```

Cells are constructed with explicit styling:
```yukino
set cell = init_cell("A", "cyan", "default", true, false, false, false, false)
```

- `fg` and `bg`: Named ANSI terminal colors (`"default"`, `"white"`, `"black"`, `"red"`, `"green"`, `"blue"`, `"cyan"`, `"yellow"`, `"magenta"`) or TrueColor HEX strings (`"#cdd6f4"`, `"#1e1e2e"`).
- The flags `bold`, `dim`, `italic`, `underline`, `reverse` map directly to standard ANSI terminal attributes.

### 2.2 Geometric Bounds: Rect
Defines the position and spatial dimensions of any visual element:

```yukino
public struct Rect {
    x: int32
    y: int32
    width: int32
    height: int32
}
```

#### Geometric Helper Functions:
- `init_rect(x, y, w, h) -> Rect`: Rectangle constructor.
- `rect_contains(r: Rect, x: int32, y: int32) -> bool`: Point-in-polygon containment test (used for mouse hit-testing).
- `rect_inner(r: Rect, pad_x: int32, pad_y: int32) -> Rect`: Computes interior bounds inset by horizontal and vertical padding.
- `rect_intersect(a: Rect, b: Rect) -> Rect`: Computes the intersection of two bounding boxes (used for scissoring).

---

## 3. Application Architecture: SiestaApp

The `SiestaApp` class orchestrates the entire application lifecycle. It manages terminal raw mode, alternate screen buffers, event polling, frame scheduling, and double-buffered rendering.

```
 +-------------------------------------------------------------------+
 |                             SiestaApp                             |
 |                                                                   |
 |  [Terminal Raw Mode] <---> [Poll Event / Read Event]              |
 |                                    |                              |
 |                                    v                              |
 |                       +-------------------------+                 |
 |                       | Key & Mouse Dispatcher  |                 |
 |                       +-------------------------+                 |
 |                         |          |           |                  |
 |                         v          v           v                  |
 |                   [Active Dialog] [Overlays] [Mounted Screen]     |
 |                         |          |           |                  |
 |                         +----------+-----------+                  |
 |                                    |                              |
 |                                    v                              |
 |                       +-------------------------+                 |
 |                       |   LayoutPass: Context   |                 |
 |                       +-------------------------+                 |
 |                                    |                              |
 |                                    v                              |
 |                       +-------------------------+                 |
 |                       |   DrawPass: BackBuffer  |                 |
 |                       +-------------------------+                 |
 |                                    |                              |
 |                                    v                              |
 |                       +-------------------------+                 |
 |                       | DoubleBuffer: Diff/Flush|                 |
 |                       +-------------------------+                 |
 |                                    |                              |
 |                                    v                              |
 |                             [Terminal ANSI]                       |
 +-------------------------------------------------------------------+
```

### 3.1 Complete Application Lifecycle in Code

The following practical example demonstrates how to construct, configure, and execute a standalone pure-Yukino application without `.spes`:

```yukino
load std.io: println
load std.itermtool.key: KeyEvent, KEY_ESC, KEY_TAB
load siesta: SiestaApp, Screen, LayoutContext, TerminalBuffer, init_cell

public class CustomDashboard extends Screen {
    public fun init() {
        super("CustomDashboard")
    }

    public fun build_layout(ref ctx: LayoutContext) {
        // Compute geometry for terminal viewport ctx.width x ctx.height
    }

    public fun draw(ref buf: TerminalBuffer) {
        // Clear background and render telemetry
        buf.draw_text(2, 1, "Monitoring Service Active", "green", "default", true, false, false, false, false)
        buf.draw_text(2, 3, "Press ESC to exit", "white", "default", false, true, false, false, false)
    }

    public fun handle_key(key: KeyEvent) -> bool {
        if key.code == KEY_ESC {
            return false // Allow default app handler to terminate loop
        }
        return true
    }
}

fun main() {
    set app = init SiestaApp(80, 24)
    app.set_auto_scale(true, 80, 24)

    set screen = init CustomDashboard()
    app.mount_screen(screen)

    // Launch event loop
    app.run()
}
```

### 3.2 SiestaApp Lifecycle Methods
- `app.init_terminal() -> bool`: Switches the terminal into raw non-canonical mode (`enable_raw_mode`), enters the alternate screen buffer (`enter_alternate_screen`), hides the cursor (`\x1b[?25l`), and activates mouse tracking (`enable_mouse`).
- `app.restore_terminal()`: Disables mouse tracking, restores cursor visibility, exits alternate screen, and restores previous terminal settings.
- `app.mount_screen(screen: Screen)`: Mounts the primary fullscreen view.
- `app.show_dialog(dialog: Dialog)`: Pushes a modal popup dialog onto the active display stack.
- `app.close_dialog()`: Closes the active dialog and restores input focus to the underlying screen.
- `app.add_overlay(overlay: Screen)`: Adds a non-modal floating overlay (such as a notification banner or tooltip layer).
- `app.remove_overlay(title: string) -> bool`: Removes an overlay by title.
- `app.check_resize() -> bool`: Queries the OS terminal size and automatically invokes `app.resize(w, h)` when dimensions change.
- `app.step() -> bool`: Executes one tick of the event loop: polls events with a 16 ms timeout, dispatches inputs, renders the frame, and returns loop continuation status.
- `app.run()`: Enters the continuous frame loop `while region this.is_running { this.step() }`.
- `app.exit()`: Sets `is_running = false`, initiating clean loop termination.

---

## 4. TerminalBuffer and Scissoring

`TerminalBuffer` represents a two-dimensional grid of cells in memory laid out as a contiguous 1D array of size `width * height`:

```yukino
public class TerminalBuffer {
    public width: int32
    public height: int32
    public cells: Cell[]
    public dirty_lines: bool[]
    public clip_stack: Rect[]
}
```

### 4.1 Indexing and Scissor Clipping
Cell index calculation uses row-major offset math:
$$\text{index} = y \cdot \text{width} + x$$

To prevent child widgets from overflowing parent boundaries, `TerminalBuffer` implements a scissor clipping stack (`clip_stack`):

```yukino
// Restrict drawing operations to a 40x10 rectangular window
buf.push_clip(init_rect(10, 5, 40, 10))

// Any draw_text call outside this rectangle is discarded automatically
buf.draw_text(0, 0, "This text will not be drawn because it is clipped")
buf.draw_text(12, 6, "This text is rendered safely inside clip bounds", "cyan")

// Restore previous clipping boundary
buf.pop_clip()
```

### 4.2 TerminalBuffer Methods
- `buf.set_cell(x: int32, y: int32, cell: Cell)`: Sets a cell if coordinates are in bounds and pass the current clip rectangle. Automatically marks row `y` as dirty (`dirty_lines[y] = true`).
- `buf.get_cell(x: int32, y: int32) -> Cell`: Retrieves the cell at the given coordinates.
- `buf.draw_text(x, y, text, fg, bg, bold, dim, italic, underline, reversed)`: Renders UTF-8 text horizontally with style attributes.
- `buf.fill(cell: Cell)`: Fills the entire buffer with a template cell (typically a space with the theme background color).
- `buf.clear()`: Resets all cells to default blank state.

---

## 5. Double Buffering (DoubleBuffer) and ANSI Optimization

To eliminate screen flickering and reduce terminal I/O bandwidth, `siesta` utilizes `DoubleBuffer`.

```
Back Buffer (rendered in memory)
  ┌───────────────┐
  │ Hello World!  │ (line 1 marked dirty)
  │ unchanged line│ (line 2 clean -> SKIPPED)
  └───────────────┘
          │
          │ compute_diff_spans(front, back)
          v
  [DiffSpan: x=0, y=0, text="Hello World!", fg="cyan"]
          │
          │ flush_diff_spans(...)
          v
  ANSI Output: "\x1b[1;1H\x1b[38;2;17;168;205mHello World!\x1b[0m"
          │
          │ swap()
          v
Front Buffer == Back Buffer
```

### 5.1 DiffSpan Coalescing Algorithm
Rather than emitting individual character escapes, `compute_diff_spans`:
1. Scans only rows where `dirty_lines[y] == true`. Clean rows that have not changed during this frame are bypassed in $O(1)$.
2. Compares `front_buffer.get_cell(x, y)` and `back_buffer.get_cell(x, y)`.
3. Merges contiguous changed cells sharing identical colors and text styles into a single `DiffSpan`.
4. The `flush_diff_spans` routine positions the terminal cursor once per span (`\x1b[Y;XH`), emits style changes only when attributes differ from the active state, and prints the entire string slice in one write.

This ensures that even at 60 FPS, terminal output consumes negligible CPU time and prevents line tearing.

---

## 6. Layout Engine and Constraint Solver

The layout subsystem computes coordinates and dimensions using formal `Constraint` rules.

### 6.1 Constraint Types
- `constraint_length(val: int32)`: Fixed character dimension.
- `constraint_percentage(pct: int32)`: Percentage of available space (1 to 100).
- `constraint_ratio(num: int32, den: int32)`: Exact rational proportion (e.g. $1/3$).
- `constraint_fill(weight: int32 = 1)`: Greedy flex fill with proportional weight (equivalent to `flex-grow`).
- `constraint_min(val: int32)`: Minimum guaranteed dimension.
- `constraint_max(val: int32)`: Maximum ceiling dimension.

### 6.2 The `solve_constraints` Solver
Takes available container space and an array of constraints, returning solved pixel/cell allocations:

```yukino
set constraints: Constraint[] = [
    constraint_length(20),     // Sidebar: 20 cells
    constraint_fill(2),        // Workspace: weight 2
    constraint_fill(1)         // Log panel: weight 1
]

set total_width: int32 = 110
set sizes = solve_constraints(total_width, ref constraints)
// sizes[0] = 20
// Remaining space = 90. Distributed proportionally:
// sizes[1] = 60 (90 * 2 / 3)
// sizes[2] = 30 (90 * 1 / 3)
```

### 6.3 Spatial Splitting: `layout_split_horizontal` and `layout_split_vertical`
Splits a bounding `Rect` into child strips with inter-element spacing `gap`:

```yukino
set bounds = init_rect(0, 0, 80, 24)
set rows = layout_split_vertical(bounds, ref [
    constraint_length(3),  // Header
    constraint_fill(1),    // Main workspace
    constraint_length(1)   // Status bar
], 0)

// rows[0] = Rect(0, 0, 80, 3)
// rows[1] = Rect(0, 3, 80, 20)
// rows[2] = Rect(0, 23, 80, 1)
```

---

## 7. Focus Navigation and Modal Management

### 7.1 FocusManager
Manages cyclic keyboard focus traversal across interactive widgets (`ButtonWidget`, `InputWidget`, `TextAreaWidget`):

```yukino
set focus_mgr = init FocusManager()

set input_node = init FocusNode("username_input", 0, false)
set button_node = init FocusNode("submit_button", 1, false)

focus_mgr.register(input_node)
focus_mgr.register(button_node)

// Move to next widget via Tab
focus_mgr.next()

// Move to previous widget via Shift+Tab
focus_mgr.prev()

// Inspect focused widget ID
if focus_mgr.get_focused_id() == "username_input" {
    // Input is currently active
}
```

### 7.2 ModalManager and Focus Traps
When a modal dialog is displayed, `ModalManager` isolates user input:
- Input events outside the dialog's bounding box are blocked.
- Pressing `Tab` cycles focus strictly among widgets registered inside the active modal entry, preventing focus escape.
- The `dim_backdrop: true` property automatically dims underlying background cells.

---

## 8. Mouse Dispatch: MouseRouter

`MouseRouter` handles hit-testing and spatial event dispatching:

```yukino
set router = init MouseRouter()

// Register clickable zones with z-index ordering
router.register(init MouseTarget("btn_cancel", init_rect(10, 15, 12, 3), 1, true))
router.register(init MouseTarget("btn_submit", init_rect(24, 15, 12, 3), 1, true))

// Dispatch click event
set result: MouseDispatchResult = router.dispatch_click(26, 16)
if result.handled {
    println("Clicked target: " + result.target_id) // "btn_submit"
}
```

- Supports mouse press, release, drag-and-drop, and scroll wheel events.
- Targets are sorted by `z_index`, ensuring modal dialogs and dropdown menus intercept clicks before background widgets.

---

## 9. Comprehensive Animation Engine

The `siesta` animation system provides an exact mathematical toolkit for creating fluid, non-blocking terminal animations.

### 9.1 Mathematical Foundations
All animations operate on normalized progress $t \in [0.0, 1.0]$:
$$t = \frac{\text{now\_ms} - \text{start\_time\_ms}}{\text{duration\_ms}}$$

The sampled property value is computed via:
$$V(t) = V_{\text{from}} + (V_{\text{to}} - V_{\text{from}}) \cdot \phi(t)$$
where $\phi(t)$ is an analytical easing curve.

### 9.2 All 10 Easing Curves
The `siesta.animation.easing` module provides the following easing functions:

1. **`linear(t)`**: Uniform, constant velocity motion.
   $$\phi(t) = t$$
2. **`ease_in_quad(t)`**: Quadratic acceleration from a standstill.
   $$\phi(t) = t^2$$
3. **`ease_out_quad(t)`**: Quadratic deceleration toward the target.
   $$\phi(t) = t \cdot (2 - t)$$
4. **`ease_in_out_quad(t)`**: Symmetrical quadratic acceleration and deceleration.
5. **`ease_in_cubic(t)`**: Cubic acceleration with pronounced inertia.
   $$\phi(t) = t^3$$
6. **`ease_out_cubic(t)`**: Cubic deceleration with a smooth landing.
   $$\phi(t) = (t - 1)^3 + 1$$
7. **`ease_in_out_cubic(t)`**: Pronounced S-curve velocity profile.
8. **`bounce_out(t)`**: Decaying bounce oscillation simulating a dropped elastic ball.
9. **`bounce_in(t)`**: Inverted bounce acceleration at start.
   $$\phi(t) = 1 - \text{bounce\_out}(1 - t)$$
10. **`bounce_in_out(t)`**: Bounce oscillation at both trajectory endpoints.
11. **`elastic_out(t)`**: Spring-like overshoot oscillation past the target point.

Curves can be dynamically sampled by name using `sample_easing(name, t)`.

### 9.3 AnimationTicker
To manage frame rates and avoid unnecessary CPU load, `SiestaApp` uses `AnimationTicker`:

```yukino
set ticker = init AnimationTicker()
ticker.init(60) // Target 60 FPS (16 ms interval)

// In an animation or game loop:
set now = 1050 // Current epoch milliseconds
if ticker.should_render(now) {
    app.render_frame()
}

// Compute sleep time before next required frame
set sleep_ms = ticker.get_sleep_time_ms(now)
```

- When all animations are inactive (`ticker.is_idle()`), the ticker suspends frame execution, allowing the process to sleep until external user input arrives.

### 9.4 Transition Primitives: Transition, Pulse, Slide

#### 1. Transition Class
Smooth scalar value interpolation (`float64`):

```yukino
// Interpolate value from 0.0 to 100.0 over 500 ms using ease_out_cubic
set anim = init Transition(0.0, 100.0, 500, current_time_ms(), "ease_out_cubic")

set current_val = anim.sample(current_time_ms())
if anim.is_finished(current_time_ms()) {
    println("Animation complete!")
}
```

#### 2. Pulse Class
Continuous triangular or periodic oscillation between minimum and maximum bounds:

```yukino
// Pulsate intensity between 0.2 and 1.0 with a 1000 ms cycle
set pulse = init Pulse(0.2, 1.0, 1000, current_time_ms())
set intensity = pulse.sample(current_time_ms())
```

#### 3. Slide Class
Two-dimensional spatial displacement from $(X_1, Y_1)$ to $(X_2, Y_2)$:

```yukino
// Slide panel in from offscreen (x=-30) into view (x=0)
set slide = init Slide(-30, 5, 0, 5, 300, current_time_ms(), "ease_out_quad")

set cur_x = slide.sample_x(current_time_ms())
set cur_y = slide.sample_y(current_time_ms())
```

---

## 10. Memory Model and Zero-Leak Architecture

Yukino does not rely on a garbage collector (GC), ensuring deterministic latency and instant response times. The `siesta` runtime is engineered to follow Yukino's memory ownership rules.

### 10.1 The `while region` Frame Loop Idiom
Interactive terminal applications run hundreds of iterations per minute. Allocating event structures, temporary strings, and diff spans on the heap would lead to memory fragmentation.

In `SiestaApp.run()`, the main loop is scoped within a `while region`:

```yukino
public fun run() {
    this.init_terminal()
    while region this.is_running {
        set running = this.step()
        if !running {
            break
        }
    }
    this.restore_terminal()
}
```

- The `region` keyword allocates an ephemeral memory arena for each frame iteration.
- All temporary objects (`Event`, `KeyEvent`, string slices, temporary span buffers) are allocated directly inside this arena.
- At the end of each frame iteration, the arena is reset in $O(1)$ without per-object destructor traversal.

### 10.2 Long-Lived State and `persistent(...)`
State that must endure across the application's entire lifetime (such as the double buffer's backing cell arrays and registered screen trees) is promoted using `persistent(...)`:

```yukino
// Screen buffer cells must outlive ephemeral frame regions
set list: Cell[] = []
for i in 0..count {
    list.push(init_cell(" ", "default", "default", false, false, false, false, false))
}
rewrite this.cells = persistent(list)
```

This guarantees:
1. Zero memory leaks over continuous 24/7 terminal service execution.
2. Zero garbage collection pauses or stuttering.
3. Minimal, deterministic memory consumption (typically 4–8 MB).

---

## 11. Navigation

- [0. Overview and Architecture](0-overview.md)
- [1. SPES Specification Language](1-spes.md)
- [2. Runtime API and Features](2-runtime.md) (current chapter)
