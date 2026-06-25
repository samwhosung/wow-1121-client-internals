# Console — the in-game console and command registry

The in-game console is the drop-down command/output screen — the backtick `` ` `` key toggles it — and the command registry behind it. It owns its **text-layout positioning math** (caret blink, line-baseline layout, caret-quad geometry, font-size scale/clamp, and mouse→line mapping), but **delegates** glyph rasterization and textured-quad drawing to the graphics device, and per-glyph metrics / font generation to the TextBlock font kernel. It lives in two `Source\Console\` translation units that bracket the sibling cvar code (see [Console variables](cvars.md)): `ConsoleClient.cpp` (the screen: line list, input-line editing, per-frame render, input/font-metric handlers) and the `ConsoleCommand` registry (parse / register / lookup / execute / history / tab-complete / help).

## The two code bands

The console code occupies two non-contiguous `.text` windows:

| Band | Range | Functions | Identity |
|---|---|---|---|
| `ConsoleClient.cpp` runtime (the screen) | `[0x638cc0, 0x63cff0)` | 96 | `__FILE__` @`0x8648c8`; `CONSOLELINE` alloc-tag @`0x864908` |
| `ConsoleCommand` registry runtime | `[0x63f880, 0x640c50)` | 30 | `.?AUCONSOLECOMMAND@@` @`0x8655c0`; `.?AV?$TSExplicitList@UCONSOLECOMMAND@@…` @`0x865494` |

The CRT static-init code of both objects sits outside these windows; its only externally-visible effect is the initial state documented below.

## The console-screen state — a global singleton

The screen is **not** a heap struct; it is a fixed block of globals. Live state sits in `.bss` (`0xc4e6xx–0xc4f8xx`), font-metric configuration in `.data` (`0x8645xx`), and tuning constants in `.rdata`.

| Global | Role |
|---|---|
| `0xc4e9ac` / `0xc4e9a8` / `0xc4e9a4` | line-list head ptr / sentinel / `TSGetLink` offset (intrusive `TSList`, newest at head) |
| `0xc4e9b4` | top-of-view / scroll-line ptr (visible-walk anchor; page = 10) |
| `0xc4e9b0` / `0xc4e668` | background / foreground(text) render-layer userdata (registered at priority `0x40c00000` / `0x40e00000`) |
| `0xc4e6f4` / `0xc4e6f8` | caret-blink accumulator (f32 seconds) / phase (0 = off, 1 = on) |
| `0xc4e848` | font path buffer `char[0x104]` (`Fonts\ARIALN.ttf`) |
| `0xc4ec34` / `0xc4ec18` | font handle (from font kernel `0x44d040`) / gx text-batch handle (from `0x5c1d60`) |
| `0xc4eac0` / `0xc4ea8c` | scale-X / scale-Y factor (`1.0`, or `1.0/range` — caret width = `2 × scaleX`) |
| `0xc4ec50` / `0xc4ec24` | char-spacing (f32 glyph-layout param) / line count (evict threshold `0x100` = 256) |
| `0xc4ead8` / `0xc4eadc` | per-line colour-table base / caret (default) colour |
| `0xc4ec10` / `0xc4ec40`·`44` | derived line-metric (`ratio · fontSize`) / scroll-fraction drag bounds |
| `0xc4ee08` | input-line buffer (command text; `ParseCommand` tokenises) |
| `0xc4f864` / `0xc4f86c` / `0xc4f84c`·`54` | command-registry hash table (base / mask) / command-list link · head |
| `0x86455c` / `0x864574` | font size (master, device units, default `0.02`; CVar-backed) / render-flags (Proportional = `0x10`) |
| `0x864544`·`48`·`4c`·`50` | text-area corners L/T/R/B = `0`/`0`/`1`/`1` (the Y-top corner doubles as a visibility gate) |
| `0x864560` / `0x864564` | derived line-height ratio / scroll value |

### `.rdata` tuning constants

| Address | Value | Use |
|---|---|---|
| `0x7ff9d8` | `1.0` | device extent / on-screen gate |
| `0x7ffd78` / `0x80679c` | `0.3` / `0.2` s | caret-blink on / off durations |
| `0x8012cc` | `0.75` | first-line baseline factor |
| `0x801360` | `0.001` | font-size CVar → device-units scale |
| `0x8029c8` / `0x8029d0` | `0.01` / `0.05` | font-size clamp range `[0.01, 0.05]` |

### Initial state

Set by the (excluded) static-init code:

- Line list self-linked empty (sentinel `0xc4e9a8`).
- Colour table `0xc4ead8..0xc4eaf8` = `ffffffff ffffffff ff808080 ffff0000 ffffff00 ffffffff ffffffff 80ffffff c0000000`.
- Two render layers registered (for 640×480 / 800×600, via `0x589900`).
- Derived line-metric `0xc4ec10` = `[0x864560] · [0x86455c]`.
- The command registry is an empty `TSExplicitList<CONSOLECOMMAND>`.

## The layout math the console owns

The unifying quantity is the **line metric**: `lineMetric = lineRatio · fontSize`, where `lineRatio = [0x864560]` and `fontSize = [0x86455c]`. It is recomputed whenever the font size or line spacing changes, and drives both the baseline ladder and the mouse→line mapping.

### Font metrics

```
font_size_set        0x639660:  fontSize = clamp(cvar · 0.001, 0.01, 0.05)   then ratio round-trip
font_metrics_reset   0x639100:  fontSize = 0.02 (default);  lineMetric = fontSize · c
line_spacing_set     0x6395c0:  lineMetric = fontSize · arg
```

`font_size_set` reads the CVar-backed master font size, multiplies by the `0.001` device scale (`0x801360`), clamps to `[0.01, 0.05]` (`0x8029c8`/`0x8029d0`), and re-derives the ratio. The font size itself defaults to `0.02` in device units.

### Caret and line baseline

```
paint_text_layout    0x63b800:  blink: accumulator += dt; toggle phase on 0.3 / 0.2 s (on/off)
                                 baseline-Y = fontSize · 0.75 + top;  next line += fontSize
                                 a line draws only while baseline-Y < 1.0 (on-screen gate)
draw_caret           0x63b990:  caret quad: width = 2 · scaleX + x;  height = baseline + fontSize
```

The caret blink is a free-running f32 accumulator of frame `dt` seconds: when the accumulator passes the current phase's duration (`0.3` s while shown, `0.2` s while hidden — `0x7ffd78`/`0x80679c`), the phase at `0xc4e6f8` toggles. The first visible line's baseline is `fontSize · 0.75 + top` (the `0.75` is `0x8012cc`); each subsequent line steps the baseline down by `fontSize`. The `< 1.0` test (`0x7ff9d8`) stops the walk at the bottom of the device-normalised viewport. Caret width is `2 · scaleX`, so it tracks the horizontal scale factor.

### Drop-down slide animation

```
slide_animate        0x63bd50:  textTop += dir · dt · rate,  clamped to the open/closed target
```

The screen slides in/out by moving the text-area top corner toward its open or closed target at a fixed rate, clamped at the endpoints.

### Scrollbar and mouse→line mapping

```
line_at_mouse        0x63c220:  ftol( (lineMetric − (1.0 − mouseY)) / fontSize )
mouse_down_drag      0x63c0b0:  scrollbar thumb hit-test + selection band
mouse_move_drag      0x63c400:  drag-follow on lineMetric
```

`line_at_mouse` inverts the baseline ladder: it converts a device-normalised mouse Y into a line index via `ftol((lineMetric − (1.0 − mouseY)) / fontSize)` (`ftol` is the C runtime float→long truncation). The scrollbar drag handlers operate against the same line metric.

### Resolution filter and scale seam

```
resolution_validate  0x63aa00:  accept a resolution only if  w / h ≥ minAspect
console_scale_init   0x63b460:  [0xc4eac0] / [0xc4ea8c] = 1.0 / range   (else 1.0)
```

`console_scale_init` establishes the X/Y scale factors at `0xc4eac0`/`0xc4ea8c` — `1.0/range` when a range is active, otherwise `1.0` — which `draw_caret` reads for its caret width.

## What it delegates

- **Graphics device** — glyph rasterization and textured-quad drawing: the gx context `0x5c1d60`, text batch (`0x5c1f00`), immediate-mode vertex submit (`0x589e30`), and render-layer registration (`0x442800`/`0x442cf0`). See [Graphics device](graphics-device.md).
- **TextBlock font kernel** — `0x44d040` (CreateFont / GenerateFont) and `0x44d340` (glyph-run layout). Per-glyph advance and kerning live here, not in the console. See [Text layout](text-layout.md).
- **Core** — the event bus and render-layer registration (`0x41fca0`). See [Core](core.md).
- **CVars** — the font and colour CVars the console registers. See [Console variables](cvars.md).
- **Storm/SMem** — allocation and string operations. **CRT/cmath** — `atof` / `ftol`.

## The command registry

The `ConsoleCommand` registry is a `TSExplicitList<CONSOLECOMMAND>` with an accompanying hash table: base at `0xc4f864`, mask at `0xc4f86c`, and the command-list link/head at `0xc4f84c`·`54`. Commands are parsed from the input-line buffer at `0xc4ee08` (`ParseCommand` tokenises the text), then looked up, executed, recorded into history, and tab-completed against this registry. Command output and errors are written to the on-screen message log handled by the system-message subsystem (see [System messages](sysmsg.md)).
