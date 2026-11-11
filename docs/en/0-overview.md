# Library siesta: Overview and Quickstart

The `siesta` library is a declarative library for creating terminal user interfaces (TUI) in the Yukino programming language. Instead of manual procedural element positioning, coordinate calculations, and character-by-character emission of ANSI escape sequences, `siesta` allows you to describe interface structures using declarative `.spes` specification files.

At build time, a procedural macro compiles this markup directly into strictly typed Yukino classes, delivering maximum execution speed, zero runtime parsing overhead, and compile-time type safety.

---

## 1. Introduction to TUI for Beginners

A Text/Terminal User Interface (TUI) is an interactive visual environment operating within the character grid of a console window.

### 1.1 How the Terminal Works at a Low Level
A standard command-line application (CLI) operates in **stream mode**: calling `println` sequentially sends strings of text to standard output (`stdout`), and the terminal emulator scrolls down as new lines arrive. In this mode, the program does not know where the cursor is on the screen and cannot move the cursor 5 rows up to update an isolated number.

A TUI application works fundamentally differently — in **2D coordinate grid mode**:
1. The terminal is treated as a rectangular matrix of character cells with fixed dimensions (for example, 120 columns by 30 rows). Each cell has `(x, y)` coordinates and stores a character, text foreground color, background color, and visual attributes (bold, dim, underline, reverse).
2. The program switches the terminal into **raw mode** using system primitives from `std.itermtool`. In raw mode:
   - Operating system line buffering is disabled: every keypress is forwarded to the application immediately (without waiting for the `Enter` key).
   - Automatic input echoing is disabled (typed characters do not appear randomly across rendered windows).
   - The **alternate screen buffer** is activated: the terminal saves the user's current shell session, presents a clean virtual screen to the TUI program, and upon application exit restores the original shell session without leaving visual artifacts.

### 1.2 The Problem with Manual Rendering (Flickering and Performance)
When implementing a TUI by hand, the classic beginner mistake looks like this:
```
Every frame (for example, 30 times per second):
1. Send full-screen clear escape sequence: \x1b[2J
2. Redraw all borders, text, and buttons from scratch via println
```
Under this approach, the screen inevitably **flickers**: the human eye perceives the momentary gap between screen clearing and the arrival of the new frame. Furthermore, transmitting tens of thousands of bytes of ANSI commands every frame wastes CPU cycles and stresses the terminal emulator's rendering pipeline.

### 1.3 How siesta Solves This: Double Buffering
The `siesta` library utilizes an industry-grade **double buffering (`DoubleBuffer`)** architecture with differential dirty-line flushing:
- Two in-memory cell buffers of identical dimensions are maintained:
  - `front_buffer`: the exact snapshot of what the user currently sees on screen.
  - `back_buffer`: an offscreen canvas where the application and widgets render the next frame.
- During rendering, the diff engine compares each line of `back_buffer` against `front_buffer`.
- Only cells that have **actually changed** since the previous frame are emitted to the terminal (dirty lines). Unmodified regions (static borders, persistent headers, background areas) are untouched.
- Adjacent cells sharing identical color attributes are coalesced into continuous ANSI spans, drastically cutting network and terminal bandwidth.
- Result: perfectly smooth, flicker-free rendering at arbitrary refresh rates.

---

## 2. Why Declarative SPES Syntax

Building a TUI procedurally in pure code leads to an explosion of layout boilerplate:
- Every panel requires manual coordinate math: `x = 2`, `y = 10`, `w = (term_w - 4) / 2`.
- When the user resizes the terminal window, all coordinates must be recalculated manually while guarding against negative numbers or boundary overflows.
- Business logic (event handlers, network calls) becomes tangled with procedural drawing code.

The **SPES (`.spes`)** syntax separates layout architecture from imperative calculations:
1. You describe the screen hierarchy as a clean tree of containers and widgets (`col`, `row`, `block`, `button`, `table`).
2. You declare responsive rules: `fill: true` (expand to take remaining space), `minWidth: 80` (hide a column if the window width drops below 80 columns), `breakpoint: "desktop"`.
3. **Zero runtime overhead**: unlike Web or Electron frameworks, `.spes` files are never parsed at application startup. The `#[siesta::res]` procedural macro reads specifications at compile time and outputs optimized native Yukino code. The application runs with the native execution speed of compiled machine code.

---

## 3. Architecture and Operational Flow

The system architecture consists of three distinct layers:

```
+----------------------------------------------------------------+
|                   1. Specification File (.spes)                |
|   theme / screen / component / dialog / shortcuts / widgets    |
+----------------------------------------------------------------+
                               |
                               v (compile-time macro siesta::res)
+----------------------------------------------------------------+
|                 2. Generated Yukino Classes                    |
|   public class MainScreen extends Screen {                     |
|       public counter: int32                                    |
|       public fun build_layout(ref ctx: LayoutContext)          |
|       public fun draw(ref buf: TerminalBuffer)                 |
|       public fun handle_key(key: KeyEvent) -> bool             |
|   }                                                            |
+----------------------------------------------------------------+
                               |
                               v (yukinc compilation + siesta_core link)
+----------------------------------------------------------------+
|                     3. siesta_core Runtime                     |
|   SiestaApp / DoubleBuffer / DiffSpans / LayoutEngine / Focus  |
+----------------------------------------------------------------+
```

### 3.1 How Data and Events Flow Between Yukino and SPES

1. **From SPES to Yukino**:
   - The macro synthesizes a native class `public class <Name> extends Screen` or dialog `extends Dialog`.
   - Containers `col` and `row` become the `build_layout(ref ctx)` method, which calculates exact `Rect` coordinates for each child node according to flex weights.
   - The widget tree becomes the `draw(ref buf: TerminalBuffer)` method, rendering elements layer by layer into the character buffer.
   - Keyboard shortcuts from the `shortcuts` block are compiled into `handle_key(key: KeyEvent) -> bool`.

2. **From Yukino to SPES**:
   - Variables declared in the `state` block become public fields on the generated class.
   - In your program, you mutate the field directly: `screen.counter = screen.counter + 1`.
   - On the next `app.render_frame()` invocation, `draw()` reads the updated state and reflects it in the back buffer.
   - The `bind: fieldName` property links input widgets (`input`, `textarea`, `checkbox`) directly to screen state without requiring manual `on_change` dispatch handlers.

---

## 4. Adding siesta to Your Project

Add the dependency to the `[libs.runtime]` section of your `yuki.toml` manifest:

```toml
[libs.runtime]
siesta = { git = "https://github.com/shareui/siesta", tag = "0.1.0", package = "siesta" }
```

### Importing Modules in Source Code
The library exports its public API through the root `siesta` module:

```yukino
load siesta: Cell, TerminalBuffer, DoubleBuffer, LayoutContext, Screen, Dialog, SiestaApp, detect_is_dark_mode
load siesta: Rect, init_rect, rect_inner, rect_intersect, LayoutNode, init_layout_node, layout_row, layout_col
load siesta: Constraint, init_constraint_length, init_constraint_percent, init_constraint_min, init_constraint_max, init_constraint_ratio
load siesta: solve_constraints, layout_split_horizontal, layout_split_vertical
load siesta: BlockWidget, ParagraphWidget, GaugeWidget, LineGaugeWidget, SparklineWidget, BarChartWidget, ScrollbarWidget, TreeWidget, TextAreaWidget
load siesta: ProgressWidget, TableWidget, TableColumn, TableRow, ListWidget, RadioGroupWidget, CheckboxWidget, ButtonWidget, Transition, Pulse, Slide
load siesta: BORDER_NONE, BORDER_PLAIN, BORDER_ROUNDED, BORDER_DOUBLE, BORDER_THICK, BORDER_ASCII
load siesta: ALIGN_LEFT, ALIGN_CENTER, ALIGN_RIGHT
load siesta: WRAP_NONE, WRAP_WORD, WRAP_CHAR
```

---

## 5. First Project: Interactive Counter with Help Dialog

Let us build a complete interactive terminal application from scratch.

### Step 1. Define the Interface (`src/views.spes`)

Create `src/views.spes`:

```spes
extern counter-app

// Define a color theme with automatic terminal dark/light background detection
theme mainTheme adaptive {
  bg: #1e1e2e
  fg: #cdd6f4
  primary: #89b4fa
  accent: #a6e3a1
  surface: #181825
}

// Modal help dialog
dialog helpDialog(title: string = "Help") {
  modal: true
  dimBackdrop: true
  width: 52
  height: 13

  shortcuts {
    "Esc": onClose
    "q": onClose
  }

  col {
    text("Application Controls:")
    text("  Space  : Increment counter by 1")
    text("  r      : Reset counter to zero")
    text("  ?      : Show this help dialog")
    text("  Esc/q  : Dismiss help or exit")
    divider
    button("Got it [Esc]")
  }
}

// Main application screen
screen mainScreen {
  state {
    count: int32 = 0
  }

  shortcuts {
    "Space": onIncrement
    "r": onReset
    "?": onHelp
  }

  col(fill: true) {
    block(border: rounded, title: "Siesta Counter Demo", titleAlign: center) {
      text("Welcome to TUI with Yukino!")
      text("Current counter value: ${this.count}")
      progress(value: 50, showPercentage: true)
      divider
      button("Click Me (Space)") {
        onClick: onIncrement
      }
    }
  }
}
```

### Step 2. Behind the Scenes: Generated Code
During compilation, the `#[siesta::res]` macro synthesizes a native Yukino class. You do not need to write this code manually, but understanding its structure is illuminating:

```yukino
public class MainScreen extends Screen {
    public count: int32
    public last_action: string
    public anim_frame: int64
    public layout_rects: Rect[]

    public fun init() {
        super("MainScreen")
        rewrite this.count = 0
        rewrite this.last_action = ""
        rewrite this.anim_frame = 0
    }

    public fun build_layout(ref ctx: LayoutContext) {
        // Computes flex geometry for col and row containers
    }

    public fun draw(ref buf: TerminalBuffer) {
        // Renders block border, title, text, and progress bar to buffer
    }

    public fun handle_key(key: KeyEvent) -> bool {
        // Dispatches Space, r, and ? key bindings
    }
}
```

### Step 3. Application Entrypoint (`src/main.ykn`)

In `src/main.ykn`, initialize the terminal, mount the screen, and run the event loop:

```yukino
load std.io: terminal_size
load std.itermtool.event: poll_event, try_read_event, Event, EVENT_KEY
load std.itermtool.key: KeyEvent, KEY_CHAR, KEY_ESC, KEY_SPACE
load siesta: SiestaApp
load .views: GeneratedViews, MainScreen, HelpDialog

// Invoke procedural code generation macro
#[siesta::res("views.spes", "locales.toml")]
class GeneratedViews {}

fun main() {
    // 1. Query initial terminal window dimensions
    set sz = terminal_size()
    set app = init SiestaApp(sz.0, sz.1)

    // 2. Instantiate screen and modal dialog
    set screen = init MainScreen()
    set help_dlg = init HelpDialog()

    // 3. Switch terminal into raw mode (hide cursor, activate alternate buffer)
    app.init_terminal()
    app.mount_screen(screen)

    // 4. Main event loop isolated in regional memory (prevents heap leaks)
    while region app.is_running {
        // Automatically detect terminal window resize events
        app.check_resize()

        // Poll for input events with a 16ms timeout (~60 FPS)
        if poll_event(16) {
            set ev = try_read_event()
            if ev != null {
                set e = ev ?? init Event(0)
                if e.kind == EVENT_KEY && e.key != null {
                    set k = e.key ?? init KeyEvent(0, "")

                    // Check if modal dialog is currently active
                    if app.is_dialog_open() {
                        if k.code == KEY_ESC || (k.code == KEY_CHAR && (k.char_val == "q" || k.char_val == "Q")) {
                            app.close_dialog()
                        } else {
                            help_dlg.handle_key(k)
                        }
                    } else {
                        // Dispatch main screen keyboard shortcuts
                        if k.code == KEY_ESC || (k.code == KEY_CHAR && (k.char_val == "q" || k.char_val == "Q")) {
                            break
                        } elif k.code == KEY_SPACE {
                            rewrite screen.count = screen.count + 1
                        } elif k.code == KEY_CHAR && (k.char_val == "r" || k.char_val == "R") {
                            rewrite screen.count = 0
                        } elif k.code == KEY_CHAR && k.char_val == "?" {
                            app.show_dialog(help_dlg)
                        } else {
                            screen.handle_key(k)
                        }
                    }
                }
            }
        }

        // Render frame with double buffer, emitting only modified cells
        app.render_frame()
    }

    // 5. Always restore the terminal to its original state before exiting
    app.restore_terminal()
}
```

### Step 4. Build and Run

```sh
# Build application
yukino build

# Run binary
./my_app
```

When started, a fullscreen interface appears with rounded borders, a progress bar, and a live counter. Pressing `Space` increments the counter instantaneously without screen flicker. Pressing `?` smoothly overlays a centered modal dialog with rounded borders and automatic dimming of the background screen. Resizing the terminal window dynamically reflows the entire layout to match the new dimensions.

---

## 6. Documentation Index

- [1. SPES Specification Language (`.spes`)](1-spes.md) — Comprehensive guide to declarative syntax: themes, components, screens, dialogs, reference of all 18 widgets, animations, flex layouts, responsiveness, and multi-file imports.
- [2. Runtime API and Features](2-runtime.md) — Reference for low-level architecture: `SiestaApp`, double buffering, dirty line tracking, focus manager, modal overlays, mouse routing, and zero-leak memory management.

---

[Next: 1. SPES Specification Language (`.spes`)](1-spes.md)
