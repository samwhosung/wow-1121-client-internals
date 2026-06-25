# World text — nameplates, floating combat text, raid icons

The `PlayerName.cpp` translation unit is the client's **overhead world-text** engine: it draws unit-name
plates, the damage/heal **floating combat text**, the **raid-target icons** that sit above units, and the
generic **world-text objects** that anchor a string to a point in the world. Each of these is a screen-space
quad whose position is derived by projecting a world anchor point through the camera every frame; the engine
owns the anchor math, the time-based fade/scale animation, and the quad/index assembly, and delegates the
actual font rasterisation, projection, and GPU submission to other subsystems.

The code lives in the address window `[0x6c63e0, 0x6c8fc0)`. Two adjacent clusters are **not** part of it
and make no calls into it: the currency COPPER/SILVER/GOLD money-format helpers `[0x6c6220, 0x6c63d7)` (over
their own `.bss` at `0xce86c4`), and the regex profanity / reserved-name filter `[0x6c8fc0, 0x6c9e78)`.

## The four object kinds

The subsystem manages four allocation-tagged record types, plus a fifth (SWING) for the melee swing-trail:

| Kind | Alloc tag | What it is |
|---|---|---|
| WTOBJECT | `0x86c5ec` | A world-text object anchored to a world point |
| PLAYERNAMEDESC | `0x86c6e0` | Per-unit descriptor holding up to 4 text slots |
| WORLDTEXTSTRING | `0x86c9c8` | One floating string (damage/heal/name), vtable `0x811314` |
| SWorldTextIcon | `0x86c994` | A raid-target icon record |
| SWING | `0x86c5d8` | A melee swing-trail segment record |

## The manager block

All state lives in one `.bss` block at roughly `0xce86d8–0xce8950`:

| Address | Contents |
|---|---|
| `0xce86d8` | WTOBJECT pool (record size `0x80`) |
| `0xce86ec` | PLAYERNAMEDESC pool (record size `0x618`) |
| `0xce880c` | WORLDTEXTSTRING pool |
| `0xce8728` / `0xce872c` | Intrusive descriptor list head / tail |
| `0xce8720` | Render flags |
| `0xce8834` / `0xce8840` | Per-text-category config table (stride `0x1c`: font / scale / alpha / timing per category) |
| `0xce87f0` / `0xce87f4` / `0xce87f8` | SWorldTextIcon ring: capacity / count / base |
| `0xce875c`+ | Anchor float constants |

A second per-category config table appears at `0xce8828` (also stride `0x1c`, four categories) feeding the
timing/scale/fade selection for the swing-trail and text categories.

### PLAYERNAMEDESC (tag `0x86c6e0`, pool record `0x618`)

| Offset | Field |
|---|---|
| `+0x10`, `+0x14` | unit GUID (low / high) |
| `+0x18` | flags |
| `+0x1c` | frame timestamp |
| `+0x20 .. +0x2c` | 4× WORLDTEXTSTRING slots |

### WORLDTEXTSTRING (tag `0x86c9c8`, vtable `0x811314`)

| Offset | Field |
|---|---|
| `+0` | vptr |
| `+8` | type |
| `+0x14 .. +0x1c` | world position |
| `+0x20` | colour |
| `+0x24` | font |
| `+0x2c` | width |
| `+0x3c` | `text[0x40]` |

The per-string alpha used by the fade animation is stored at byte `this[0x23]`.

`SWorldTextIcon` (tag `0x86c994`) records have stride `0x18` and are held in an SArray (the icon ring above).

## Lifecycle and entry points

**Submit (the hot path) — `0x6c7840`.** This is the damage/heal floating-text submit API, driven from the
object-layer CGUnit_C world-object update (`0x607140` / `0x6072e0` / `0x6128b0`). It formats the string,
dedups it against the current target, then calls `0x6c73f0` to resolve the GUID to a `CGObject` and append it
to that unit's PLAYERNAMEDESC (each desc holds up to 4 WORLDTEXTSTRING slots).

**Create descriptor — `0x6c76a0`.** Called from CGUnit (`0x60701c`); allocates a PLAYERNAMEDESC from the pool
at `0xce86ec`, stores the unit GUID, and links it into the intrusive descriptor list
(`0xce8728` head / `0xce872c` tail).

**Render — `0x6c7900` / `0x6c7950`.** The per-frame render entries (called from the UI layer at `0x482d70` /
`0x482e7a`) drive `0x6c8820`, the render driver, which locks a GxPrimitive vertex buffer and builds the
world-text quads. The per-descriptor render is `0x6c7cc0` / `0x6c80b0`, which projects each anchor through the
world→screen helper `0x483ee0` and reads the per-category descriptor config (`0xce8834`, stride `0x1c`).

**Init / shutdown.** Init `0x6c7470` (from `_initterm` at `0x401625`) registers the CVars and the per-frame
update callback `0x6c75c0` via `0x63db90`. Shutdown is `0x6c7630` + `0x6c8520`.

The melee swing-trail is its own small machine: init `0x6c6750` → callback `0x6c67f0` → step `0x6c6560`.

## The anchor and animation math

These are the closed-form computations the engine owns; everything else (font measurement, projection, GPU
submission) is delegated.

**Anchor lift — `0x6c73f0`.** The text anchor is the unit position raised by one-third of a unit:

```
z' = unit_z − 1/3        ; fsub [0x807a40]
```

**Distance-based plate scale — `0x6c6e90`.** Name plates fade their scale with camera distance `d`, then
build a billboarded atlas-cell corner-quad:

```
scale = d > 4 ? (d / 4) · 1.5 · 0.2 : 0.2
```

**Swing-trail cadence — `0x6c6560`.** Given elapsed time and a segment count:

```
n        = floor(elapsed · (1/300) · segcount)
step     = segcount / max(n, 1)
```

**Animation parameter — `0x6c8070`.** A normalised progress, computed from an unsigned elapsed time, an
i32 duration, and an f32 span (the result is left on the FP stack at runtime, not stored):

```
param = (unsigned elapsed) / dur_i32 · span_f32
```

**Keyframe interpolation — `0x6c80b0`** (the early-out return is at `0x6c819d`). Either an affine map or, for
category 2, a 3-segment piecewise curve read from the keyframe table at `0x8112dc`; the result is clamped to
a floor of `0.001`.

**Time-based alpha fade — `0x6c82e0`.** A fade-in / opaque-middle / fade-out envelope. The endpoints scale a
unit progress `t` by `255` or `127`, then convert with `__ftol` and clamp; the result is written to the
per-string alpha byte `this[0x23]`.

## Quad assembly

**Label clip-quad — `0x6c7cc0`.** Projects the world anchor to screen, clamps to the screen rect, emits the
four quad corners, and computes a midpoint scaled by `0.5`.

**Icon glyph quad — `0x6c7f10`.** Pushes a raid-icon quad offset to `x = −40`, `y = 16` (the source uses the
`·0.0` zeroing idiom followed by `fsubp`); the icon texture is selected from the table at `0xce88dc` indexed
by `type − 5`.

**Render driver — `0x6c8820`.** Builds the text-quad geometry: vertex positions are `xyz · 32` at vertex
stride `0x14`, the index buffer per quad is `{0, 1, 3, 3, 1, 2}`, and the icon colour is packed as
`(b << 24) | 0xffffff`.

**Ortho matrix — `0x6c8d00`.** Takes the screen rect, computes `(r − l, b − t)`, runs them through `__ftol`,
multiplies by the device scale, offsets by `±0.5`, and builds the orthographic projection via the gxumath
helper `0x5c3d90`.

**String measurement — `0x6c81a0`** measures via the gx font system and applies a small floating-point computation tail;
the companion `0x6c82c0` is a width-dirty guard that stores only on a `!=` change.

## What is delegated, and the runtime-only piece

The render driver and clip path wrap the GPU and font systems: GxPrimitive vertex-buffer lock/draw, the
gxumath ortho helper `0x5c3d90` (see [Rendering math](rendering-math.md)), C math, the world→screen
projection `0x483ee0` (see [Camera](camera.md)), and the UI layer (see [UI](ui.md)). Font measurement and
glyph rasterisation come from the text system (see [Text layout](text-layout.md)).

The fully composed on-screen render — the exact pixels produced by `0x483ee0` plus the GPU device plus a
fully-loaded world/object/font state — depends on live runtime state and is not reproducible from a closed
formula; it is observed at runtime rather than computed here. The computations the engine performs directly (anchor lift, fade/scale,
keyframe interpolation, quad/index assembly, ortho matrix) is the part the engine computes deterministically.
