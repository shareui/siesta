# SPES Specification Language (.spes)

The **SPES (`.spes`)** language is a declarative domain-specific language (DSL) for defining terminal user interfaces with the `siesta` library in Yukino. It is engineered to separate interface hierarchy, styling, theme definitions, and layout mechanics from application logic.

During project compilation, the procedural macro `#[siesta::res]` parses `.spes` source files, resolves the dependency graph, validates types, and compiles declarative layout trees directly into native, strongly typed Yukino classes (`Screen`, `Dialog`, reusable components, and data structures). This ensures maximum execution speed with zero runtime overhead.

---

## 1. File Structure and Syntax

A canonical `.spes` specification file consists of a module header with a unique identifier, an import block, theme palettes, and view definitions:

```spes
// 1. Unique module identifier across the build graph
extern main-dashboard

// 2. Importing external modules by their symbolic identifiers
use overview-screen(*)
use settings-screen(*)
use auth-forms(LoginForm, RegisterModal)

// 3. Adaptive color theme definition
theme catppuccinDark adaptive {
  bg: #1e1e2e
  fg: #cdd6f4
  primary: #89b4fa
  surface: #181825
}

// 4. Fullscreen view declaration
screen dashboardScreen {
  col(fill: true) {
    text("Siesta Control Center") {
      fg: theme.primary
      bold: true
    }
  }
}
```

### 1.1 Directive `extern <identifier>`
Every `.spes` file must begin with an `extern` declaration. This defines the system-wide symbolic module identifier used by other files to import its views:

- **Naming Conventions**: The parser accepts any valid identifier (`kebab-case`, `snake_case`, `camelCase`, `PascalCase`). The canonical recommended convention for `.spes` modules is `kebab-case` (`dashboard-views`, `auth-forms`, `network-telemetry`).
- **Graph Construction and Collision Detection**: At compile time, the macro scans all `.spes` files and constructs an identifier registry `SpesRegistry`. If two different files declare the same `extern` identifier, compilation halts immediately with a fatal error:
  `conflicting module identifier 'foo' defined in both 'path/a.spes' and 'path/b.spes'`.

### 1.2 Single-Line Comments `//`
SPES supports single-line comments. Everything following `//` up to the end of the line is ignored by the lexer:

```spes
// Metric panel configuration
row(height: 3) {
  // Output active network state
  text("Network active")
}
```

### 1.3 Directive `use <module>(...)`
The `use` directive links the current specification file with components, screens, and dialogs declared in other `.spes` files. Resolution is performed **strictly by the `extern` identifier**, independent of physical disk filenames:

```spes
// Import all public entities from telemetry-views
use telemetry-views(*)

// Explicitly import specific entities
use user-profiles(ProfileCard, UserAvatar)
use security-modals(ConfirmPurgeDialog)
```

- An asterisk `*` imports all screens, dialogs, and components from the target module.
- Comma-separated identifiers restrict visibility to specific exported symbols.

### 1.4 String Localization `tr(...)`
Rather than hardcoding string literals, SPES includes native internationalization support. Any string wrapped with `tr("key")` is indexed by the compiler and looked up in the active translation catalog:

```spes
screen loginScreen {
  col {
    text(tr("auth.welcome_message"))
    input(placeholder: tr("auth.enter_password")) {
      password: true
    }
  }
}
```

Localization dictionaries are located under `locales/<lang>.json` in key-value JSON format. The macro generates optimized dictionary lookups, and runtime language switching is performed via `app.set_locale("en")`.

---

## 2. Color System and Theming

The `siesta` rendering subsystem supports standard 16-color ANSI terminal palettes as well as 24-bit TrueColor (16.7 million colors).

### 2.1 Supported Color Formats
1. **Named ANSI Colors**:
   - `"default"` (retains terminal default color without ANSI override);
   - `"black"`, `"white"`, `"red"`, `"green"`, `"blue"`, `"cyan"`, `"yellow"`, `"magenta"`.
2. **TrueColor HEX Literals**:
   - Hexadecimal strings such as `"#1e1e2e"`, `"#cdd6f4"`, `"#89b4fa"`.
   - At runtime, HEX codes are translated into direct RGB ANSI escape sequences `\x1b[38;2;R;G;Bm` (foreground) and `\x1b[48;2;R;G;Bm` (background).

### 2.2 Theme Declarations (`theme`) and Adaptive Contrast (`adaptive`)
Themes define color tokens for UI widgets:

```spes
theme dracula adaptive {
  bg: #282a36
  fg: #f8f8f2
  primary: #bd93f9
  surface: #44475a
  accent: #ff79c6
}
```

- The `adaptive` modifier instructs the compiler to generate a dual-mode palette.
- When launched on Linux or Windows, the runtime queries terminal environment variables (`COLORFGBG`, `TERM_THEME`, `COLOR_SCHEME`) using `detect_is_dark_mode()`.
- If running inside a light terminal emulator, the adaptive theme automatically inverts dark background colors into contrasted light hues (`#eff1f5`), preventing unreadable black-on-black text.

### 2.3 Applying Theme Tokens
Any widget color attribute accepts theme token references using the `theme.` prefix:

```spes
block(title: "Resource Monitor") {
  fg: theme.primary
  bg: theme.surface
  text("Threads running") {
    fg: theme.accent
  }
}
```

---

## 3. Core Visual Containers

SPES defines three top-level container primitives: `screen`, `dialog`, and `component`. Each compiles to a native Yukino class with explicit lifecycle methods.

### 3.1 Screen: `screen`
A fullscreen view occupying the entire terminal viewport. A screen inherits from `Screen`:

```spes
screen overviewScreen {
  state {
    requestCount: int32 = 0
    serviceName: string = "API Gateway"
  }

  shortcuts {
    "Ctrl+R": onReload
    "q": onQuit
  }

  col(fill: true) {
    text("Service: " + serviceName)
    button("Reload") {
      onClick: onReload
    }
  }
}
```

#### Screen Anatomy and Lifecycle:
1. **Inheritance**: The generated class is declared as `public class OverviewScreen extends Screen`.
2. **Initialization (`init()`)**: The constructor initializes base class state with `super("OverviewScreen")`, resets animation ticker state `anim_frame = 0`, allocates layout rectangles, and assigns initial `state` values.
3. **Layout Pass (`build_layout(ref ctx: LayoutContext)`)**: Called upon terminal resize or prior to redrawing. Solves flex constraints for all child nodes and populates `this.layout_rects`.
4. **Drawing Pass (`draw(ref buf: TerminalBuffer)`)**: Increments `this.anim_frame`, clears the background with theme background color `buf.fill(...)`, and draws children into the terminal buffer respecting active scissor clips.
5. **Key Dispatch (`handle_key(key: KeyEvent) -> bool`)**: Matches key events against the `shortcuts` table, executes matching handler actions, and returns `true` if consumed.

### 3.2 Modal Dialog: `dialog`
A centered popup window rendered above the active screen. Dialogs inherit from `Dialog`:

```spes
dialog confirmDeleteDialog(itemName: string) {
  modal: true
  dimBackdrop: true
  width: 50
  height: 12

  shortcuts {
    "Esc": onCancel
    "Enter": onConfirm
  }

  col(fill: true) {
    text("Delete Confirmation") {
      bold: true
      fg: "red"
    }
    paragraph("Are you sure you want to delete " + itemName + "? This action cannot be undone.")
    row {
      button("Cancel") { onClick: onCancel }
      button("Delete") { onClick: onConfirm }
    }
  }
}
```

#### Dialog Behavior and Configuration:
- `modal: true`: Captures focus and input events. Keyboard and mouse events are routed exclusively to the dialog and prevented from reaching underlying screens.
- `dimBackdrop: true`: The runtime dims all cells outside the dialog boundaries by marking `dim = true` on buffer cells.
- `width` and `height`: Fixed character grid dimensions.
- **Centering Calculation**: The runtime automatically computes `dx = (term_w - size_w) / 2` and `dy = (term_h - size_h) / 2` and draws a decorative `╭───╮` border.

### 3.3 Reusable Component: `component`
A modular UI building block parameterized with props and optional local state:

```spes
component userCard(name: string, role: string, isActive: bool = true) {
  block(border: rounded) {
    row {
      text(name) { bold: true }
      text(if isActive { " [Online]" } else { " [Offline]" }) {
        fg: if isActive { "green" } else { "white" }
      }
    }
    text("Role: " + role) { dim: true }
  }
}
```

- Props defined in parentheses become constructor parameters for `init(...)`.
- Components compile into standalone classes with their own `build_layout` and `draw` methods.
- Components can declare local state via `state { ... }`.

---

## 4. Reactive State (`state`) and Two-Way Binding (`bind:`)

The `state` block declares reactive instance fields with explicit Yukino types and default values:

```spes
screen accountSettingsScreen {
  state {
    email: string = "user@domain.org"
    subscribeNewsletter: bool = true
    preferredServer: int32 = 0
  }

  col {
    text("Account Preferences")
    
    // Two-way binding for string inputs
    input(placeholder: "Enter email address") {
      bind: email
    }

    // Two-way boolean binding for checkboxes
    checkbox("Subscribe to newsletter updates") {
      bind: subscribeNewsletter
    }

    // Two-way integer selection binding for radio groups
    radiogroup(vertical: true) {
      bind: preferredServer
    }
  }
}
```

### Two-Way Binding Mechanics:
1. During compilation, the macro creates accessor methods for each bound field: `get_email() -> string`, `set_email(v: string)`.
2. On render, the widget retrieves the current state value directly from the screen instance.
3. When the user interacts with the widget (typing into `InputWidget` or toggling `CheckboxWidget`), event handlers immediately update the backing field (`this.email` or `this.subscribeNewsletter`).
4. Developers do not need to write manual buffer synchronization logic.

---

## 5. Keyboard Shortcuts (`shortcuts`)

The `shortcuts` block provides declarative key binding:

```spes
shortcuts {
  "Ctrl+S": onSave
  "Ctrl+Q": onQuit
  "F5":     onRefresh
  "Tab":    onFocusNext
  "Esc":    onDismiss
  "Enter":  onSubmit
  "Space":  onToggle
  "Up":     onPrevious
  "Down":   onNext
}
```

The macro compiles this mapping into an efficient dispatcher:
```yukino
public fun handle_key(key: KeyEvent) -> bool {
    if key.ctrl && key.char_val == "s" {
        this.trigger_onSave()
        rewrite this.last_action = "onSave"
        return true
    }
    if key.code == KEY_ESC {
        this.trigger_onDismiss()
        rewrite this.last_action = "onDismiss"
        return true
    }
    return false
}
```

---

## 6. Declarative Flex Layout

Layout in `siesta` follows a block flexbox model based on `col` (vertical column) and `row` (horizontal row) containers:

```spes
col(fill: true) {
  // Fixed-height top navigation bar
  row(height: 3) {
    text("Logo")
    text("User: admin")
  }

  // Flexible middle section with sidebar and workspace
  row(fill: true, gap: 1) {
    col(width: 25) {
      text("Navigation Sidebar")
    }
    col(fill: true) {
      text("Main Workspace")
    }
  }

  // Fixed-height footer status bar
  row(height: 1) {
    text("Status: Connected")
  }
}
```

### Container Attributes for `col` and `row`:
- `fill: true`: Container greedily expands to absorb available free space (equivalent to `flex-grow: 1`). Space is distributed proportionally when multiple children have `fill: true`.
- `width` / `height`: Explicit cell dimensions.
- Percentage sizes: String values such as `width: "50%"`, `height: "20%"`.
- `padding`, `paddingX`, `paddingY`: Inner cell margins.
- `gap`: Cell spacing between adjacent children.
- `justify`: Primary axis alignment (`start`, `center`, `end`).
- `align`: Cross axis alignment (`stretch`, `start`, `center`, `end`).

### Layout Utilities: `divider` and `spacer`
- `divider`: Horizontal separator line rendered across available width with `─`.
- `spacer`: Expanding element with `fill: true` that pushes subsequent siblings to the container edge.

---

## 7. Responsive Breakpoints

Terminal interfaces running in different environments (narrow sidebars at 60 columns vs wide monitors at 160 columns) require responsive adaptation:

```spes
col(fill: true) {
  // Hide directory tree when terminal width is below 90 columns
  tree(minWidth: 90)

  // Hide micro-chart on compact screens
  sparkline(hideBelow: 75)

  // Render extended telemetry only on wide desktop terminals
  block(breakpoint: "desktop") {
    text("Extended Diagnostics")
  }
}
```

### Breakpoint Constraints:
- `minWidth` / `min_w`: Minimum terminal columns required for rendering.
- `minHeight` / `min_h`: Minimum terminal rows required.
- `maxWidth` / `max_w`: Maximum terminal columns for display.
- `maxHeight` / `max_h`: Maximum terminal rows.
- `hideBelow`: Alias for `minWidth`.
- `breakpoint`: Named screen category:
  - `"desktop"`: width $\ge 120$ cells.
  - `"compact+"`: width $\ge 80$ cells.
  - `"compact"`: width between $80$ and $119$ cells.
  - `"mobile"`: width $< 80$ cells.

---

## 8. Declarative Animations (`anim:`)

Dynamic visual indicators and status updates can be animated declaratively:

```spes
col {
  text("Syncing with cloud...") {
    anim: pulse
    fg: "yellow"
  }

  progress(value: 80) {
    anim: slide
    fg: "cyan"
  }
}
```

### Built-in Animation Directives:
1. `anim: pulse`: Periodically alternates text emphasis by toggling `bold` based on `(this.anim_frame % 20 < 10)`.
2. `anim: slide`: Smoothly offsets widget horizontal position using `+ cast(this.anim_frame % 8, "int32")`.
3. **Frame Clock Integration**: Screens automatically increment `this.anim_frame` on each `draw(...)` invocation. No background threads or timers are spawned, ensuring predictable deterministic terminal rendering.

---

## 9. Comprehensive Specification of All 18 Widgets

Each widget has a formal specification, SPES properties, and a backing class in `core.widgets`.

---

### 9.1 `text`
Single-line formatted text with full UTF-8 Unicode support.

- **Runtime Class**: `TextWidget` (`libs/core/core/widgets/text.ykn`).
- **Purpose**: Static labels, titles, computed expressions, and status readouts.
- **SPES Properties**:
  - First argument: String content (`"Hello"`, `tr("key")`, or state expression).
  - `fg`: Foreground color (`"default"`, `"cyan"`, `"#cdd6f4"`).
  - `bg`: Background color (`"default"`, `"#1e1e2e"`).
  - `bold: bool`: Bold font weight.
  - `dim: bool`: Faint / dim intensity.
  - `italic: bool`: Italic font style.
  - `underline: bool`: Single underline.
  - `reverse: bool`: Inverted foreground and background.
  - `anim`: Animation directive (`pulse`, `slide`).
- **Example**:
  ```spes
  text("Server listening on port 8080") {
    fg: "green"
    bold: true
  }
  ```

---

### 9.2 `block`
Structural container box with border outlines, title headers, and padding.

- **Runtime Class**: `BlockWidget` (`libs/core/core/widgets/block.ykn`).
- **Purpose**: Grouping related widgets into panels, cards, and sections.
- **SPES Properties**:
  - `border`: Border style (`rounded`, `plain`, `double`, `thick`, `ascii`, `none`).
    - `rounded`: Rounded box corners `╭ ╮ ╰ ╯`.
    - `plain`: Single light lines `┌ ┐ └ ┘`.
    - `double`: Double lines `╔ ╗ ╚ ╝`.
    - `thick`: Bold lines `┏ ┓ ┗ ┛`.
    - `ascii`: Clean ASCII fallback `+ - | +`.
    - `none`: Invisible bounds without borders.
  - `title: string`: Header text embedded into the top border.
  - `titleAlign`: Title alignment (`left`, `center`, `right`).
  - `fg`, `bg`: Border and content colors.
  - `padding`: Interior inset margin.
- **Example**:
  ```spes
  block(border: rounded, title: "Cluster Diagnostics", titleAlign: center) {
    fg: "cyan"
    text("Active Nodes: 12")
  }
  ```

---

### 9.3 `paragraph`
Multiline text block with word wrapping and alignment modes.

- **Runtime Class**: `ParagraphWidget` (`libs/core/core/widgets/paragraph.ykn`).
- **Purpose**: Displaying documentation, license agreements, help text, and error logs.
- **SPES Properties**:
  - First argument: Text string.
  - `wrap`: Word wrapping algorithm:
    - `word`: Wraps text at whitespace boundaries without breaking words.
    - `char`: Hard-wraps at container column width.
    - `none`: Truncates lines exceeding available width.
  - `align`: Text alignment (`left`, `center`, `right`).
  - `fg`, `bg`: Text and background colors.
- **Example**:
  ```spes
  paragraph("Yukino compiles into native machine code without garbage collector overhead.") {
    wrap: word
    align: left
    fg: "white"
  }
  ```

---

### 9.4 `button`
Interactive button with role styles and focus/hover states.

- **Runtime Class**: `ButtonWidget` (`libs/core/core/widgets/button.ykn`).
- **Purpose**: Triggering actions, submitting dialog forms, navigating views.
- **SPES Properties**:
  - First argument: Button label text.
  - `role`: Semantic visual role:
    - `default`: Neutral frame `[ Label ]`.
    - `primary`: Accent action color (cyan/blue).
    - `success`: Affirmative action (green).
    - `warning`: Warning caution (yellow).
    - `danger`: Destructive action (red).
  - `onClick`: Action handler identifier.
  - `disabled: bool`: Disables interaction and dims colors.
  - `fg`, `bg`: Color overrides.
- **Example**:
  ```spes
  button("Deploy Changes", role: primary) {
    onClick: onDeploy
  }
  ```

---

### 9.5 `input`
Single-line text input field with cursor indicator and password masking.

- **Runtime Class**: `InputWidget` (`libs/core/core/widgets/input.ykn`).
- **Purpose**: Text entry, usernames, search queries, API keys.
- **SPES Properties**:
  - `placeholder: string`: Hint text shown when empty.
  - `password: bool`: Masks characters with asterisks `*`.
  - `bind`: Name of a string field in `state` for two-way synchronization.
  - `fg`, `bg`: Text and field background colors.
- **Example**:
  ```spes
  input(placeholder: "Enter API Token") {
    bind: apiTokenField
    password: true
  }
  ```

---

### 9.6 `textarea`
Multiline text editor with line numbers gutter and cursor positioning.

- **Runtime Class**: `TextAreaWidget` (`libs/core/core/widgets/textarea.ykn`).
- **Purpose**: Editing code snippets, markdown documents, configuration scripts.
- **SPES Properties**:
  - `showLineNumbers: bool`: Displays left gutter with line numbers `1 |`, `2 |`.
  - `height: int32`: Fixed vertical view height.
  - `bind`: Backing multiline string state variable.
  - `fg`, `bg`: Editor text and background colors.
- **Example**:
  ```spes
  textarea(showLineNumbers: true, height: 15) {
    bind: scriptBuffer
  }
  ```

---

### 9.7 `checkbox`
Boolean toggle option with `[x]` / `[ ]` glyphs.

- **Runtime Class**: `CheckboxWidget` (`libs/core/core/widgets/selection.ykn`).
- **Purpose**: Enabling or disabling binary configuration flags.
- **SPES Properties**:
  - First argument: Label text.
  - `checked: bool`: Initial toggle state.
  - `bind`: Backing `bool` field in `state`.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  checkbox("Enable TLS 1.3 Encryption", checked: true) {
    bind: tlsEnabled
  }
  ```

---

### 9.8 `radiogroup`
Radio button group for mutually exclusive single choice selection.

- **Runtime Class**: `RadioGroupWidget` (`libs/core/core/widgets/selection.ykn`).
- **Purpose**: Selecting run modes, output encodings, themes.
- **SPES Properties**:
  - `vertical: bool`: Orientation (vertical or horizontal).
  - `bind`: Backing `int32` index field in `state`.
  - `fg`, `bg`: Label colors.
- **Example**:
  ```spes
  radiogroup(vertical: true) {
    bind: selectedLogFormatIndex
  }
  ```

---

### 9.9 `list`
Virtualized scrollable list of text items.

- **Runtime Class**: `ListWidget` (`libs/core/core/widgets/list.ykn`).
- **Purpose**: File lists, event logs, task queues.
- **SPES Properties**:
  - `visibleRows: int32`: Windowing viewport size.
  - `fg`, `bg`: Text and highlight colors.
- **Example**:
  ```spes
  list(visibleRows: 12) {
    fg: "white"
  }
  ```

---

### 9.10 `table`
Multi-column tabular data grid with column headers and text alignment.

- **Runtime Class**: `TableWidget` (`libs/core/core/widgets/table.ykn`).
- **Purpose**: Database results, system process tables, financial ledger data.
- **SPES Properties**:
  - `id`: Unique identifier for data binding in Yukino.
  - `width`, `height`: Grid dimensions.
- **Example**:
  ```spes
  table(id: processListTable) {
    width: "100%"
    height: 10
  }
  ```

---

### 9.11 `progress`
Horizontal progress bar with fractional sub-cell Unicode rendering.

- **Runtime Class**: `ProgressWidget` (`libs/core/core/widgets/progress.ykn`).
- **Purpose**: Visualizing file download completion, build status, long jobs.
- **Sub-Cell Precision**: Uses `fractional_block` with 8 Unicode fractions (`▏`, `▎`, `▍`, `▌`, `▋`, `▊`, `▉`, `█`) for smooth sub-cell fills.
- **SPES Properties**:
  - `value: int32`: Progress value (0 to 100).
  - `showPercentage: bool`: Displays percentage string `[ 75% ]`.
  - `fg`, `bg`: Fill and track colors.
- **Example**:
  ```spes
  progress(value: 84, showPercentage: true) {
    fg: "cyan"
  }
  ```

---

### 9.12 `gauge`
Two-dimensional block meter with centered percentage and title label.

- **Runtime Class**: `GaugeWidget` (`libs/core/core/widgets/gauge.ykn`).
- **Purpose**: CPU usage, memory utilization, disk volume meters.
- **SPES Properties**:
  - `percent: float64`: Normalized fill ratio (0.0 to 1.0).
  - `label: string`: Centered label text.
  - `height: int32`: Gauge block height.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  gauge(percent: 0.68, label: "RAM: 10.8 GB / 16.0 GB", height: 3) {
    fg: "yellow"
  }
  ```

---

### 9.13 `linegauge`
Compact single-row fractional progress meter.

- **Runtime Class**: `LineGaugeWidget` (`libs/core/core/widgets/gauge.ykn`).
- **Purpose**: Embedding fill meters into tight status bars and headers.
- **SPES Properties**:
  - `ratio: float64`: Fill ratio (0.0 to 1.0).
  - `label: string`: Accompanying text.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  linegauge(ratio: 0.92, label: "Connection Pool") {
    fg: "red"
  }
  ```

---

### 9.14 `sparkline`
Micro-chart trendline using vertical Unicode block glyphs.

- **Runtime Class**: `SparklineWidget` (`libs/core/core/widgets/sparkline.ykn`).
- **Purpose**: Real-time trend visualization for latency, throughput, and CPU load.
- **Density Characters**: Uses 8 vertical Unicode levels: ` `, `▂`, `▃`, `▄`, `▅`, `▆`, `▇`, `█`.
- **SPES Properties**:
  - `max: int32`: Maximum value normalization scale.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  sparkline(max: 100) {
    fg: "green"
  }
  ```

---

### 9.15 `barchart`
Multi-column bar chart for comparative series metrics.

- **Runtime Class**: `BarChartWidget` (`libs/core/core/widgets/barchart.ykn`).
- **Purpose**: Displaying request frequencies, response time distributions.
- **SPES Properties**:
  - `max: int32`: Scale ceiling.
  - `width: int32`: Character width per bar.
  - `gap: int32`: Character gap between bars.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  barchart(max: 100, width: 4, gap: 2) {
    fg: "blue"
  }
  ```

---

### 9.16 `scrollbar`
Scrollbar track and thumb widget for vertical or horizontal navigation.

- **Runtime Class**: `ScrollbarWidget` (`libs/core/core/widgets/scrollbar.ykn`).
- **Purpose**: Indicating scroll position and viewport ratio in long views.
- **SPES Properties**:
  - `total: int32`: Total content height.
  - `viewport: int32`: Viewport window size.
  - `position: int32`: Current scroll offset.
- **Example**:
  ```spes
  scrollbar(total: 500, viewport: 25, position: 120)
  ```

---

### 9.17 `tabs`
Horizontal tab navigation bar for switching views.

- **Runtime Class**: `TabsWidget` (`libs/core/core/widgets/tabs.ykn`).
- **Purpose**: Multi-page settings, dashboard tabs, view modes.
- **SPES Properties**:
  - `activeIndex: int32`: Zero-based active tab index.
  - `fg`, `bg`: Colors.
- **Example**:
  ```spes
  tabs(activeIndex: 0) {
    fg: "cyan"
  }
  ```

---

### 9.18 `canvas`
High-resolution vector and raster canvas powered by Braille 2x4 dot matrices.

- **Runtime Class**: `CanvasWidget` (`libs/core/core/widgets/canvas.ykn`).
- **Purpose**: Drawing geometric lines, curves, functions, and diagrams.
- **Braille Resolution**: Each terminal character cell hosts a 2 horizontal by 4 vertical dot matrix ($2 \times 4$). An $80 \times 24$ cell terminal provides a $160 \times 96$ pixel graphics canvas.
- **SPES Properties**:
  - `width: int32`: Cell width.
  - `height: int32`: Cell height.
  - `fg`, `bg`: Pixel colors.
- **Example**:
  ```spes
  canvas(width: 60, height: 12) {
    fg: "green"
  }
  ```

---

## 10. Comprehensive Example: Cloud Infrastructure Dashboard

The following complete `dashboard.spes` file demonstrates modular imports, custom themes, components, dialogs, responsive breakpoints, and animations:

```spes
extern cloud-monitor

use auth-forms(UserBadge)

theme oceanicDark adaptive {
  bg: #1b2b34
  fg: #d8dee9
  primary: #6699cc
  surface: #101a20
  accent: #99c794
}

component metricTile(caption: string, value: string, isOk: bool = true) {
  block(border: rounded) {
    text(caption) { dim: true }
    text(value) {
      bold: true
      fg: if isOk { theme.accent } else { "red" }
    }
  }
}

screen monitorScreen {
  state {
    activeTab: int32 = 0
    searchQuery: string = ""
    isLiveUpdate: bool = true
  }

  shortcuts {
    "Ctrl+Q": onQuit
    "Ctrl+R": onRefresh
    "F1": onShowHelp
  }

  col(fill: true) {
    // Header navigation
    row(height: 3) {
      text("Cloud Infrastructure Monitor") {
        bold: true
        fg: theme.primary
      }
      spacer
      text("Status: Active") {
        anim: pulse
        fg: theme.accent
      }
    }

    divider

    // Metric tiles row
    row(height: 5, gap: 1) {
      metricTile("CPU Utilization", "24.5%", true)
      metricTile("Message Queue", "1,420", true)
      metricTile("HTTP Errors", "0.02%", true)
    }

    // Main workspace
    row(fill: true, gap: 1) {
      // Sidebar search column
      col(width: 30) {
        text("Filter Pods:")
        input(placeholder: "Pod name prefix...") {
          bind: searchQuery
        }
        checkbox("Live streaming (1 sec)", checked: true) {
          bind: isLiveUpdate
        }
      }

      // Main content table
      col(fill: true) {
        tabs(activeIndex: 0)
        table(id: clusterTable) {
          width: "100%"
          height: 10
        }
      }
    }

    // Bottom status bar
    row(height: 1) {
      text("Ready | F1 Help | Ctrl+Q Exit") { dim: true }
    }
  }
}
```

---

## 11. Navigation

- [0. Overview and Architecture](0-overview.md)
- [1. SPES Specification Language](1-spes.md) (current chapter)
- [2. Runtime API and Features](2-runtime.md)
