# Рантайм API и возможности библиотеки siesta

Модуль `siesta_core` — это высокопроизводительное рантайм-ядро для терминальных интерфейсов на языке Yukino. В то время как макрос `#[siesta::res]` компилирует разметку `.spes`, именно `siesta_core` берет на себя всю низкоуровневую работу: взаимодействие с экраном терминала, двойную буферизацию, расчет геометрии flex-контейнеров, кольцевую навигацию фокуса, обработку событий мыши и управление памятью.

---

## 1. Подключение: load siesta

Все публичные структуры, классы, алгоритмы компоновки и виджеты рантайма импортируются через единый фасад `siesta`:

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

## 2. Базовые структуры данных: Cell и Rect

### 2.1 Знакоместо экрана: Cell
Каждая точка на экране терминала представлена структурой `Cell`. Это неделимый квант терминальной графики, содержащий один символ UTF-8 и его визуальные параметры:

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

Конструктор создает ячейку с заданными параметрами:
```yukino
set cell = init_cell("A", "cyan", "default", true, false, false, false, false)
```

- `fg` и `bg`: именованные терминальные цвета (`"default"`, `"white"`, `"black"`, `"red"`, `"green"`, `"blue"`, `"cyan"`, `"yellow"`, `"magenta"`) либо TrueColor HEX-строки (`"#cdd6f4"`, `"#1e1e2e"`).
- Флаги `bold`, `dim`, `italic`, `underline`, `reverse` соответствуют базовым ANSI-атрибутам терминала.

### 2.2 Двумерный прямоугольник: Rect
Определяет положение и размер любого элемента интерфейса:

```yukino
public struct Rect {
    x: int32
    y: int32
    width: int32
    height: int32
}
```

#### Вспомогательные функции геометрии:
- `init_rect(x, y, w, h) -> Rect`: конструктор прямоугольника.
- `rect_contains(r: Rect, x: int32, y: int32) -> bool`: проверка попадания точки внутрь прямоугольника (используется для hit-тестирования мыши).
- `rect_inner(r: Rect, pad_x: int32, pad_y: int32) -> Rect`: вычисление внутреннего прямоугольника с отступами.
- `rect_intersect(a: Rect, b: Rect) -> Rect`: вычисление области пересечения двух прямоугольников (используется для ограничения области отсечения scissoring).

---

## 3. Архитектура приложения SiestaApp

Класс `SiestaApp` является диспетчером всего приложения. Он управляет сырым режимом терминала, альтернативным экраном, вводом, циклом кадров и двойной буферизацией.

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

### 3.1 Полный цикл приложения в коде

Ниже приведен практический пример создания, настройки и запуска приложения:

```yukino
load std.io: println
load std.itermtool.key: KeyEvent, KEY_ESC, KEY_TAB
load siesta: SiestaApp, Screen, LayoutContext, TerminalBuffer, init_cell

public class CustomDashboard extends Screen {
    public fun init() {
        super("CustomDashboard")
    }

    public fun build_layout(ref ctx: LayoutContext) {
        // Вычисление геометрии для терминала ctx.width x ctx.height
    }

    public fun draw(ref buf: TerminalBuffer) {
        // Очистка и отрисовка данных
        buf.draw_text(2, 1, "Мониторинг запущен", "green", "default", true, false, false, false, false)
        buf.draw_text(2, 3, "Нажмите ESC для выхода", "white", "default", false, true, false, false, false)
    }

    public fun handle_key(key: KeyEvent) -> bool {
        if key.code == KEY_ESC {
            return false // Позволить приложению обработать выход
        }
        return true
    }
}

fun main() {
    set app = init SiestaApp(80, 24)
    app.set_auto_scale(true, 80, 24)

    set screen = init CustomDashboard()
    app.mount_screen(screen)

    // Запуск главного цикла
    app.run()
}
```

### 3.2 Методы управления приложением SiestaApp
- `app.init_terminal() -> bool`: переводит терминал в сырой неканонический режим (`enable_raw_mode`), переключается на альтернативный буфер экрана (`enter_alternate_screen`), скрывает текстовый курсор (`\x1b[?25l`) и включает поддержку мыши (`enable_mouse`).
- `app.restore_terminal()`: отключает мышь, возвращает видимость курсора, покидает альтернативный экран и восстанавливает оригинальные настройки терминала.
- `app.mount_screen(screen: Screen)`: монтирует главное полноэкранное представление.
- `app.show_dialog(dialog: Dialog)`: открывает модальное всплывающее окно.
- `app.close_dialog()`: закрывает текущий диалог и восстанавливает фокус на основном экране.
- `app.add_overlay(overlay: Screen)`: добавляет немодальный слой поверх основного экрана (например, плавающее уведомление или панель подсказок).
- `app.remove_overlay(title: string) -> bool`: удаляет слой по его названию.
- `app.check_resize() -> bool`: проверяет, изменился ли физический размер окна терминала, и при необходимости автоматически вызывает `app.resize(w, h)`.
- `app.step() -> bool`: выполняет одну итерацию главного цикла: опрашивает события клавиатуры и мыши с таймаутом 16 мс, диспетчеризирует их, вызывает `render_frame()` и возвращает статус работы приложения.
- `app.run()`: входит в непрерывный цикл `while region this.is_running { this.step() }`.
- `app.exit()`: устанавливает флаг `is_running = false`, инициируя завершение главного цикла.

---

## 4. Буфер терминала TerminalBuffer и Scissoring

`TerminalBuffer` представляет собой двумерную матрицу ячеек в памяти, развернутую в непрерывный одномерный массив размера `width * height`:

```yukino
public class TerminalBuffer {
    public width: int32
    public height: int32
    public cells: Cell[]
    public dirty_lines: bool[]
    public clip_stack: Rect[]
}
```

### 4.1 Адресация и scissoring (отсечение)
Адрес ячейки вычисляется по формуле построчного смещения:
$$\text{index} = y \cdot \text{width} + x$$

Для предотвращения выхода дочерних виджетов за границы своих контейнеров буфер реализует стек областей отсечения (`clip_stack`):

```yukino
// Ограничить область рисования окном 40x10
buf.push_clip(init_rect(10, 5, 40, 10))

// Все операции draw_text вне этого прямоугольника автоматически игнорируются
buf.draw_text(0, 0, "Этот текст не будет нарисован, так как он вне клипа")
buf.draw_text(12, 6, "Этот текст виден внутри клипа", "cyan")

// Снять ограничение
buf.pop_clip()
```

### 4.2 Основные методы TerminalBuffer
- `buf.set_cell(x: int32, y: int32, cell: Cell)`: устанавливает символ и стили ячейки, если координаты находятся внутри границ буфера и не отсечены текущим клипом. При изменении автоматически помечает строку `y` как грязную (`dirty_lines[y] = true`).
- `buf.get_cell(x: int32, y: int32) -> Cell`: считывает ячейку по заданным координатам.
- `buf.draw_text(x, y, text, fg, bg, bold, dim, italic, underline, reversed)`: выводит строку с автоматическим декодированием UTF-8 символов.
- `buf.fill(cell: Cell)`: заливает весь буфер указанной ячейкой (обычно пробелом с фоновым цветом темы).
- `buf.clear()`: быстрая очистка содержимого буфера.

---

## 5. Двойная буферизация DoubleBuffer и оптимизация ANSI

Для полного устранения мерцания терминала и минимизации трафика вывода в `siesta` реализован класс `DoubleBuffer`.

```
Back Buffer (формируется в памяти)
  ┌───────────────┐
  │ Hello World!  │ (строка 1 помечена dirty)
  │ unchanged line│ (строка 2 чистая -> ПРОПУСКАЕТСЯ)
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

### 5.1 Алгоритм слияния DiffSpan
Вместо отправки каждого символа отдельной escape-последовательностью алгоритм `compute_diff_spans`:
1. Сканирует только те строки, для которых `dirty_lines[y] == true`. Чистые строки, не изменившиеся за текущий кадр, мгновенно пропускаются.
2. Поэлементно сравнивает `front_buffer.get_cell(x, y)` и `back_buffer.get_cell(x, y)`.
3. Соседние изменившиеся ячейки с одинаковыми цветами и атрибутами объединяются в непрерывный отрезок `DiffSpan`.
4. Функция `flush_diff_spans` позиционирует курсор терминала один раз в начало каждого отрезка (`\x1b[Y;XH`), включает только изменившиеся стили оформления и печатает весь текст блока одной операцией.

Благодаря этому вывод даже на частоте 60 кадров в секунду потребляет доли процента процессорного времени и исключает разрывы строк.

---

## 6. Движок раскладки и вычислитель ограничений

Движок раскладки `siesta` вычисляет размеры и координаты элементов на основе ограничений `Constraint`.

### 6.1 Типы ограничений (Constraint)
- `constraint_length(val: int32)`: фиксированный размер в символах.
- `constraint_percentage(pct: int32)`: процент от общей доступной длины (от 1 до 100).
- `constraint_ratio(num: int32, den: int32)`: точная рациональная пропорция (например, $1/3$).
- `constraint_fill(weight: int32 = 1)`: жадное заполнение свободного пространства с весовым коэффициентом (аналог `flex-grow`).
- `constraint_min(val: int32)`: минимальный гарантированный размер.
- `constraint_max(val: int32)`: максимальное ограничение размера.

### 6.2 Функция `solve_constraints`
Функция принимает доступную длину контейнера и массив ограничений, возвращая массив вычисленных размеров:

```yukino
set constraints: Constraint[] = [
    constraint_length(20),     // Боковая панель 20 ячеек
    constraint_fill(2),        // Центральная область (вес 2)
    constraint_fill(1)         // Панель логов (вес 1)
]

set total_width: int32 = 110
set sizes = solve_constraints(total_width, ref constraints)
// sizes[0] = 20
// Оставшееся место = 90. Распределяется по весам:
// sizes[1] = 60 (90 * 2 / 3)
// sizes[2] = 30 (90 * 1 / 3)
```

### 6.3 Разделение областей: `layout_split_horizontal` и `layout_split_vertical`
Нарезают исходный прямоугольник `Rect` на дочерние полосы с учетом промежуточного зазора `gap`:

```yukino
set bounds = init_rect(0, 0, 80, 24)
set rows = layout_split_vertical(bounds, ref [
    constraint_length(3),  // Шапка
    constraint_fill(1),    // Рабочая область
    constraint_length(1)   // Подвал
], 0)

// rows[0] = Rect(0, 0, 80, 3)
// rows[1] = Rect(0, 3, 80, 20)
// rows[2] = Rect(0, 23, 80, 1)
```

---

## 7. Подсистема навигации фокуса и модальных окон

### 7.1 FocusManager
Управляет кольцевой навигацией клавиатурного фокуса между интерактивными виджетами (`ButtonWidget`, `InputWidget`, `TextAreaWidget`):

```yukino
set focus_mgr = init FocusManager()

set input_node = init FocusNode("username_input", 0, false)
set button_node = init FocusNode("submit_button", 1, false)

focus_mgr.register(input_node)
focus_mgr.register(button_node)

// Переход к следующему элементу по Tab
focus_mgr.next()

// Переход к предыдущему элементу по Shift+Tab
focus_mgr.prev()

// Проверка активности
if focus_mgr.get_focused_id() == "username_input" {
    // Ввод активен
}
```

### 7.2 ModalManager и фокус-ловушки
При открытии диалога `ModalManager` изолирует клавиатурный и мышиный ввод:
- События вне прямоугольника диалога блокируются.
- Клавиша `Tab` циклически перемещает фокус только между полями ввода внутри активного диалога, не позволяя фокусу выскочить на фоновый экран.
- Опция `dim_backdrop: true` затеняет фоновые ячейки.

---

## 8. Маршрутизация событий мыши: MouseRouter

`MouseRouter` обеспечивает высокоточную диспетчеризацию событий указателя:

```yukino
set router = init MouseRouter()

// Регистрация интерактивных зон с учетом z-index
router.register(init MouseTarget("btn_cancel", init_rect(10, 15, 12, 3), 1, true))
router.register(init MouseTarget("btn_submit", init_rect(24, 15, 12, 3), 1, true))

// Обработка клика
set result: MouseDispatchResult = router.dispatch_click(26, 16)
if result.handled {
    println("Клик по элементу: " + result.target_id) // "btn_submit"
}
```

- Поддерживает события нажатия кнопки, отпускания, перетаскивания (drag-and-drop) и вращения колеса мыши.
- Сортирует цели по `z_index`, гарантируя, что модальные окна и всплывающие меню перехватывают клики первыми.

---

## 9. Полноценная подсистема анимаций

Библиотека `siesta` предоставляет законченный математический аппарат для создания плавных терминальных анимаций без подвисаний и скачков.

### 9.1 Математические основы
Все анимации строятся на нормализованном параметре времени $t \in [0.0, 1.0]$:
$$t = \frac{\text{now\_ms} - \text{start\_time\_ms}}{\text{duration\_ms}}$$

Значение интерполируемой величины вычисляется по формуле:
$$V(t) = V_{\text{from}} + (V_{\text{to}} - V_{\text{from}}) \cdot \phi(t)$$
где $\phi(t)$ — функция плавности (easing function).

### 9.2 Все 10 функций плавности (Easing Curves)
В модуле `siesta.animation.easing` реализованы следующие аналитические кривые:

1. **`linear(t)`**: равномерное прямолинейное движение.
   $$\phi(t) = t$$
2. **`ease_in_quad(t)`**: квадратичное ускорение из состояния покоя.
   $$\phi(t) = t^2$$
3. **`ease_out_quad(t)`**: квадратичное замедление к концу пути.
   $$\phi(t) = t \cdot (2 - t)$$
4. **`ease_in_out_quad(t)`**: квадратичный разгон и последующее плавное торможение.
5. **`ease_in_cubic(t)`**: кубическое ускорение с подчеркнутой инерцией.
   $$\phi(t) = t^3$$
6. **`ease_out_cubic(t)`**: кубическое торможение с мягкой остановкой.
   $$\phi(t) = (t - 1)^3 + 1$$
7. **`ease_in_out_cubic(t)`**: выраженный кубический S-образный профиль скорости.
8. **`bounce_out(t)`**: эффект упругого отскока шарика от пола с затухающей амплитудой.
9. **`bounce_in(t)`**: затухающие колебания в начале движения.
   $$\phi(t) = 1 - \text{bounce\_out}(1 - t)$$
10. **`bounce_in_out(t)`**: отскоки в начале и в конце фазы перехода.
11. **`elastic_out(t)`**: затухающее колебание пружины с вылетом за пределы целевой точки.

Выборка по строковому имени производится функцией `sample_easing(name, t)`.

### 9.3 Тактовый генератор AnimationTicker
Для контроля частоты кадров и предотвращения 100% нагрузки на процессор рантайм использует `AnimationTicker`:

```yukino
set ticker = init AnimationTicker()
ticker.init(60) // Целевая частота 60 кадров/сек (интервал 16 мс)

// В игровом или анимационном цикле:
set now = 1050 // миллисекунды текущего времени
if ticker.should_render(now) {
    app.render_frame()
}

// Оптимальное время сна для таймера/селектора
set sleep_ms = ticker.get_sleep_time_ms(now)
```

- Если нет активных анимаций (`ticker.is_idle()`), тикер сообщает приложению о переходе в режим ожидания ввода без лишней перерисовки экрана.

### 9.4 Классы переходов: Transition, Pulse, Slide

#### 1. Класс Transition
Плавная интерполяция скалярной величины (`float64`):

```yukino
// Переход значения от 0.0 до 100.0 за 500 мс с кривой ease_out_cubic
set anim = init Transition(0.0, 100.0, 500, current_time_ms(), "ease_out_cubic")

set current_val = anim.sample(current_time_ms())
if anim.is_finished(current_time_ms()) {
    println("Анимация завершена!")
}
```

#### 2. Класс Pulse
Непрерывные синусоидальные/треугольные колебания между минимальным и максимальным значениями:

```yukino
// Пульсация яркости от 0.2 до 1.0 с периодом 1000 мс
set pulse = init Pulse(0.2, 1.0, 1000, current_time_ms())
set intensity = pulse.sample(current_time_ms())
```

#### 3. Класс Slide
Двумерное геометрическое перемещение окна, диалога или курсора из точки $(X_1, Y_1)$ в $(X_2, Y_2)$:

```yukino
// Выезд панели из-за левого края (x=-30) в рабочую позицию (x=0)
set slide = init Slide(-30, 5, 0, 5, 300, current_time_ms(), "ease_out_quad")

set cur_x = slide.sample_x(current_time_ms())
set cur_y = slide.sample_y(current_time_ms())
```

---

## 10. Управление памятью и модель нулевых утечек

Язык Yukino не использует тяжелый сборщик мусора (GC), гарантируя предсказуемое время отклика в терминальных интерфейсах. Рантайм `siesta` спроектирован в строгом соответствии с канонами управления памятью Yukino.

### 10.1 Идиома `while region` в главном цикле
В интерактивных приложениях терминал выполняет сотни и тысячи кадров в минуту. Если создавать структуры событий и временные строковые буферы в глобальной куче, произойдет фрагментация памяти.

В `SiestaApp.run()` главный цикл оформлен через конструкцию `while region`:

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

- Ключевое слово `region` открывает локальную арену памяти на одну итерацию кадра.
- Все временные объекты кадра (объекты `Event`, `KeyEvent`, срезы строк, временные векторы `spans` при diff-расчете) выделяются в быстром региональном пуле.
- В конце каждой итерации цикла арена мгновенно очищается за $O(1)$ без вызова деструкторов для каждого отдельного байта.

### 10.2 Долгоживущие данные и функция `persistent(...)`
Данные, которые должны жить на протяжении всей работы приложения (массив ячеек буфера экрана, зарегистрированные экраны, дерево оверлеев), маркируются через вызов встроенной функции `persistent(...)`:

```yukino
// Буфер экрана должен сохраняться между кадрами
set list: Cell[] = []
for i in 0..count {
    list.push(init_cell(" ", "default", "default", false, false, false, false, false))
}
rewrite this.cells = persistent(list)
```

Такое четкое разделение гарантирует:
1. Абсолютно нулевые утечки памяти при суточной непрерывной работе приложения.
2. Отсутствие пауз и подвисаний на сборку мусора, свойственных интерфейсам на Java, Python или Node.js.
3. Фиксированное и минимальное потребление оперативной памяти (обычно менее 4-8 МБ).

---

## 11. Навигация по документации

- [0. Обзор библиотеки и архитектура](0-overview.md)
- [1. Декларативный язык SPES](1-spes.md)
- [2. Рантайм API и возможности](2-runtime.md) (текущий раздел)
