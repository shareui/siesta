# Siesta: declarative TUI library

A declarative library for building terminal user interfaces (TUI) for the [Yukino](https://github.com/yukitsuri/yukino) programming language.

`siesta` allows developers to define terminal user interfaces in clean, declarative `.spes` specification files. At compile time, a procedural macro compiles the layout into strictly-typed Yukino screen, component, and dialog classes with themes, localization, adaptive layout, and input event routing. Zero manual layout boilerplate, zero runtime parsing overhead.

---

## Features

- **Declarative SPES Grammar (`.spes`)**: Expressive, clean layout syntax for screens, reusable components, modal dialogs, shortcuts, and visual themes.
- **Modularity and Multi-File Support**: Decompose large interfaces into discrete screen files using `use filename(*)`.
- **Compile-Time Code Generation**: The `#[siesta::res("views.spes", "locales.toml")]` procedural macro generates native Yukino classes (`Screen`, `Dialog`, components), eliminating string parsing at runtime.
- **Rich Widget Collection (18 widgets)**:
  - Text & containers: `text`, `paragraph` (word/char wrapping), `block` (borders and titles), `divider`.
  - Controls: `button`, `input`, `textarea` (multiline code/text editor with line numbers), `checkbox`, `radio` / `radiogroup`, `tabs`.
  - Data & lists: `list` (virtualized scrolling), `table` (multi-column tables), `tree` (hierarchical explorer with branches).
  - Indicators & analytics: `progress` (percentage bars), `gauge`, `lineGauge`, `sparkline` (Unicode micro-charts), `barchart` (bar diagrams), `scrollbar`.
  - High-res graphics: `canvas` (high-definition 2x4 braille dot matrix canvas).
- **Flex Layout and Constraints**: Automatic computation for `col` and `row` containers with proportional weights (`fill: true`), fixed lengths, percentages, and min/max limits.
- **Responsive Sizing & Dynamic Hiding**: Built-in `minWidth`, `minHeight`, `maxWidth`, `maxHeight`, `hideBelow`, and breakpoint ranges (`mobile`, `compact`, `compact+`, `desktop`) automatically adapt to small terminal dimensions without buffer overflow.
- **Themes & Adaptive Dark/Light Detection**: Static color palettes and adaptive themes (`theme myTheme adaptive`) with automated cross-platform terminal background detection for Linux and Windows (`detect_is_dark_mode()`).
- **Strict Type-Safe Localization (i18n)**: Translation catalog in `locales.toml`, compile-time resolution via `tr"..."`, default `en` locale, and automated 3-tier fallback chain (`requested -> en -> key identifier`).
- **Declarative Animations**: Frame-driven effects (`anim: pulse`, `anim: slide`) defined directly in widget attributes.
- **High-Performance Rendering**: Double buffering (`DoubleBuffer`), dirty-line tracking, clipping rectangles (scissoring), and streaming ANSI state coalescence without flickering.
- **Strict Memory Safety**: Zero frame-level heap leakage, regional allocation pools, and deterministic resource release.

---

## Installation

Add `siesta` to your project manifest under the `[libs.runtime]` section in `yuki.toml`:

```toml
[libs.runtime]
siesta = { git = "https://github.com/shareui/siesta", tag = "0.1.0", package = "siesta" }
```

---

## Quick Example

### 1. Define your UI (`src/views.spes`)

Create a `src/views.spes` file:

```spes
extern app-views

theme darkTheme adaptive {
  bg: #1e1e2e
  fg: #cdd6f4
  primary: #89b4fa
  accent: #a6e3a1
}

screen mainScreen {
  state {
    counter: int32 = 0
  }

  shortcuts {
    "Space": onIncrement
    "q": onQuit
  }

  col(fill: true) {
    block(border: rounded, title: "Siesta Demo", titleAlign: center) {
      text("Declarative TUI with Yukino")
      text("Counter: ${this.counter}")
      progress(value: 65, showPercentage: true)
      button("Click Me (Space)") {
        onClick: onIncrement
      }
    }
  }
}
```

### 2. What the macro generates behind the scenes

The `#[siesta::res("views.spes", "locales.toml")]` macro transforms the `.spes` specification into native Yukino classes:

```yukino
public class MainScreen extends Screen {
    public counter: int32

    public fun init() {
        super("MainScreen")
        rewrite this.counter = 0
    }

    public fun build_layout(ref ctx: LayoutContext) {
        // Automatically computed flex container geometry
    }

    public fun draw(ref buf: TerminalBuffer) {
        // Double-buffered widget rendering
    }

    public fun handle_key(key: KeyEvent) -> bool {
        // Keyboard shortcut routing
    }
}
```

### 3. Application entrypoint (`src/main.ykn`)

```yukino
load std.io: terminal_size
load std.itermtool.event: poll_event, try_read_event, Event, EVENT_KEY
load std.itermtool.key: KeyEvent, KEY_CHAR, KEY_ESC
load siesta: SiestaApp
load .views: GeneratedViews, MainScreen

#[siesta::res("views.spes", "locales.toml")]
class GeneratedViews {}

fun main() {
    set sz = terminal_size()
    set app = init SiestaApp(sz.0, sz.1)
    set screen = init MainScreen()

    app.init_terminal()
    app.mount_screen(screen)

    while region app.is_running {
        app.check_resize()

        if poll_event(16) {
            set ev = try_read_event()
            if ev != null {
                set e = ev ?? init Event(0)
                if e.kind == EVENT_KEY && e.key != null {
                    set k = e.key ?? init KeyEvent(0, "")
                    if k.code == KEY_ESC || (k.code == KEY_CHAR && k.char_val == "q") {
                        break
                    }
                    screen.handle_key(k)
                }
            }
        }

        app.render_frame()
    }

    app.restore_terminal()
}
```

---

## Documentation

- **English Documentation**:
  - [Overview and Quickstart](docs/en/0-overview.md) — TUI fundamentals, architecture, comparison with procedural rendering, and linking Yukino with SPES.
  - [SPES Specification Language (`.spes`)](docs/en/1-spes.md) — Themes, screens, modal dialogs, reference of all 18 widgets, animations, modularity, and i18n localization.
  - [Runtime API and Features](docs/en/2-runtime.md) — Public Yukino classes, double buffering, layout solving, responsiveness, input events, and zero-leak memory management.

- **Russian Documentation**:
  - [Обзор и быстрый старт](docs/ru/0-overview.md) — архитектура, концепция, сравнение с процедурным подходом и связь между Yukino и SPES.
  - [Язык спецификаций SPES (`.spes`)](docs/ru/1-spes.md) — темы, экраны, модальные диалоги, справочник всех 18 виджетов, анимации, модульность и локализация.
  - [Рантайм API и возможности](docs/ru/2-runtime.md) — публичные классы Yukino, двойная буферизация, раскладка, адаптивность, события и управление памятью.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 shareui.
